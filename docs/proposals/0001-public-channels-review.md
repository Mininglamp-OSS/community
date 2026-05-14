# RFC-0001: Public-in-Space Channels — Engineering Review

- **Authors**: Octo engineering team
- **Reviewed RFC version**: v0.1 (the draft attached to tracking issue [octo-server#23](https://github.com/Mininglamp-OSS/octo-server/issues/23))
- **Date**: 2026-05-14
- **Code baseline**: `octo-server` `develop`
- **Companion document**: [`./0001-public-channels.md`](./0001-public-channels.md) (the v0.2 design that incorporates the recommendations here)

---

## TL;DR / Outcome

The original v0.1 design called for "lazy resolution of subscribers via the WuKongIM `IDatasource.GetSubscribers` callback". After end-to-end source-reading of the WuKongIM upstream and `octo-server` modules, this turns out to be a load-bearing misunderstanding: the delivery-time subscriber list is read from WuKongIM's own pebble-backed `wkdb` store, *not* from any datasource callback. Every existing eager pattern in the OCTO codebase (notably the thread / 子区 feature) confirms this — subscribers are written into wkdb explicitly via `IMAddSubscriber` / `IMRemoveSubscriber`, and the `IDatasource` callback path is not consulted on the hot path.

The recommended path forward is **eager fanout**: every visibility flip and every Space-membership change must explicitly enumerate the affected channel set and write the corresponding subscriber deltas. This is structurally identical to how OCTO's thread feature already operates in production, so the architectural risk is low — but it changes the implementation profile materially. The original effort estimate of ~11.5 person-days underestimated the work; the realistic estimate is **~21 person-days**.

A secondary discovery from the same review is that the session-list path (`api_conversation.go: ExistMembers`) hard-filters on the presence of an explicit `group_member` row. Without a `group_member` row a public channel will not appear in the session list even if WuKongIM has already pushed its messages. The original v0.1 noted this as one of three candidate solutions ("(a) lazy `is_implicit` insertion") but treated it as an open question; in fact it is the only viable option, and the cleanup of `is_implicit` rows must be exhaustive across **five** distinct trigger paths or implicit access leaks past Space-membership changes.

The review additionally identified three further blind spots (bot-ownership semantics under implicit subscription, the Forbidden-mute interaction with Space-admin posting, the migration `ALGORITHM` / `LOCK` clauses) and confirmed that the SQL injection / cross-Space isolation / channel-name-XSS surfaces are tractable with documented mitigations.

**Outcome**: v0.2 incorporates all recommended changes. This report is preserved alongside the proposal as the engineering provenance for the design's structural choices.

---

## Scope of this review

The original tracking issue asked for a verdict on ten specific design questions. The sections that follow address each in order, plus five blind-spot findings (§3) that the v0.1 draft did not anticipate, plus a security audit of the resulting design (§9).

The review consisted of reading:

- The full v0.1 RFC draft (linked from the tracking issue body).
- 13 files across `octo-server`'s `modules/group`, `modules/thread`, `modules/message`, `modules/webhook`, and `pkg/space`.
- 5 files of WuKongIM upstream covering the delivery path, store layer, wkdb layer, and the `IDatasource` interface.

No production tracing was performed; the conclusions in §5 (the delivery-path investigation) are derived from static call-graph analysis of the upstream source. A 30-minute confirming probe is included as an optional addendum, but its result is already determined by the static analysis.

---

## 1. §4 `ResolveGroupSubscribers` SQL performance under large Spaces

### Verdict

The SQL itself is fine. **The query is rarely exercised on the hot path** because, as established in §5, message delivery does not call into `octo-server`'s webhook datasource. The function survives only as a helper for administrative APIs and as a fallback for the few non-delivery code paths that still consult the datasource.

In other words: performance is not the binding constraint. The binding constraint is **correctness of the eager-fanout writes** that drive WuKongIM's wkdb (see §3 blind-spot #1).

### Quantitative estimate (for the rare-call paths)

```sql
SELECT sm.uid FROM space_member sm
INNER JOIN space s ON s.space_id = sm.space_id AND s.status = 1
WHERE sm.space_id = ? AND sm.status = 1
UNION
SELECT gm.uid FROM group_member gm
WHERE gm.group_no = ? AND gm.is_deleted = 0 AND gm.status = 1
```

- 1 000-member Space + 50 explicit external members → ~1 050-row result set; both subqueries hit indexes; `UNION DISTINCT` does a tempfile sort but at this scale runs in 5–15 ms.
- 10 000-member Space → tempfile sort at 50–150 ms; per-instance QPS ceiling around 50–100.
- ≤ 1 000 members → < 5 ms; comfortably within budget.

### Recommended change

For future scaling beyond 10 000-member Spaces, switch to `UNION ALL + NOT IN`:

```sql
SELECT DISTINCT uid FROM (
  SELECT sm.uid FROM space_member sm
    JOIN space s ON s.space_id = sm.space_id AND s.status = 1
    WHERE sm.space_id = ? AND sm.status = 1
  UNION ALL
  SELECT gm.uid FROM group_member gm
    WHERE gm.group_no = ? AND gm.is_deleted = 0 AND gm.status = 1
      AND gm.uid NOT IN (
        SELECT uid FROM space_member WHERE space_id = ? AND status = 1
      )
) t
```

`UNION ALL` plus an explicit anti-join lets the optimiser hash-merge instead of falling back to a tempfile sort. v0.2 adopts this form for the helper.

This is, however, a footnote: the real bottleneck is the fanout architecture covered in §5.

---

## 2. §7 Q1–Q7 — verdict on each open question

### Q1 — Who can flip `visibility`? (`creator + manager`)

**Agreed.** This matches the existing channel-rename / dissolve / archive permission model and avoids the failure mode of a Space admin unilaterally publishing someone else's private channel.

**Additional requirement**: every flip writes an immutable audit row (`group_visibility_change`) recording operator, from-value, to-value, timestamp. Customer-support and compliance queries — *"when did this channel become public?"* — depend on having a ground-truth log; reconstructing this from message-history side-effects after the fact is fragile.

### Q2 — `private → public` historical-message visibility (= invisible to new subscribers)

**Agreed**, but the v0.1 draft did not specify the implementation mechanism. Three candidates were considered:

- **(a) `channel_offset` lower bound** — chosen. Reuses the table OCTO already maintains for the `allow_view_history_msg = 0` invite path. Reference: `octo-server/modules/message/event.go:413-477 updateMembersChannelOffset` writes per-user offsets to `maxSeq` for new joiners of channels with `allow_view_history_msg = 0`. The exact same code shape applies to `visibility 0 → 1`.
- **(b) `IDatasource.ChannelInfo` returns `visible_seq_min`** — rejected. WuKongIM's `ChannelInfo` protocol does not define such a field, requiring an upstream patch. Even if upstream accepted it, a per-message-per-user filter is `O(messages × subscribers)` overhead, untenable at 1 000-member-Space delivery volume. The `channel_offset` approach pushes the filter to the client-side sync path, which is already designed for per-user starting points.
- **(c) Per-user `join_seq` column on `group_member`** — rejected. Adds a column whose only purpose is to duplicate what `channel_offset` already records. Worse, it conflates two distinct semantics into one row: `group_member.uid` should mean "this user is a member of the channel", and the moment a user *became* a subscriber is properly a sync-state property, not a membership property.

**v0.2 specifies (a)** with the same write semantics as the existing `allow_view_history_msg = 0` path. Strong-form rule: when `visibility` is flipped, the `allow_view_history_msg` per-channel switch is treated as `0` for the new implicit subscribers regardless of its configured value, ensuring the visibility transition is always a hard cutover rather than a leaky one.

### Q3 — `public → private` close: wipe to the explicit roster

**Agreed in outcome, but the original justification was weak.** The v0.1 draft's reasoning was *"option (a) snapshot-and-freeze is implementation-complex and the UI is hard to express"*. That reasoning falls over: if Q6 is implemented as lazy `is_implicit` insertion, then by the time `1 → 0` runs, the implicit `group_member` rows already exist and the snapshot would just be `UPDATE ... SET is_implicit = 0`. Implementation complexity is comparable.

The **strong** reason is **social contract**. A user subscribed to a `visibility = 1` channel under the mental model *"I am in this Space, so I see this channel"* — not *"I am a named member of this group"*. Promoting them to explicit membership at close time creates a new class of complaint (*"I never joined this channel; why can't I leave it?"*). The wipe is the only outcome aligned with the way users described the feature when they signed up to it.

**Required mitigations**:

- The wipe must batch `IMRemoveSubscriber` calls; the WuKongIM wkdb does not auto-purge subscribers when their `group_member` row is deleted (see §5).
- The audit row in `group_visibility_change` must record the affected uid list — operations needs the data when responding to "I lost access" tickets.
- The operator's UI must show a strong confirmation (*"This will revoke channel access for approximately N Space members"*) before committing the flip.

### Q4 — External groups can be public: yes

**Agreed.** §6 of the v0.1 draft and Q4 are mutually consistent because `managerAdd` already enforces `is_external = 0` (`octo-server/modules/group/db.go:107-114`); external members can never become managers, therefore they can never flip visibility. This deliberately means external members cannot publicise a channel.

**Required clarification**: the §6 permission matrix should explicitly state that *visibility-flip operators must be internal members*, so that future readers don't infer an external-manager loophole. The companion v0.2 carries this note.

**Operational note**: `visibility = 1, is_external_group = 1` is a valid combination (an internal Space's public channel that also has external collaborators). The UI must double-tag these channels (*"Public · External members"*) so operators are not surprised that external collaborators can read messages in what looks like a Space-internal channel.

### Q5 — Leaving a Space invalidates public-channel access

**Agreed**, but the implementation language in v0.1 was misleading. The draft said *"Subscribers callback resolves on every delivery"*. As established in §5, that callback is not invoked on delivery. The actual mechanism must be:

```
On space_member.{remove, status 1→0}:
  enumerate all visibility=1 channels of the Space
  for each channel:
    DELETE group_member rows where uid=X AND is_implicit=1
    IMRemoveSubscriber(channelID, [X])
    for each non-deleted thread:
      IMRemoveSubscriber(threadChannelID, [X])
```

This is not "automatic invalidation" — it is **active, event-driven invalidation**. The downstream consequence: the message-history that the user already received remains in their local cache (acceptable), but `ExistMembers` filters out the channel from their next sync (so the channel disappears from the session list, as expected).

### Q6 — Conversation state (`last_read` / `mute` / `stick` / `unread`) — the largest blind spot

The v0.1 draft listed three options and labelled (a) lazy `is_implicit` as the favourite. **(a) is not a preference; it is the only viable option.** Reason: the session-list path enforces `ExistMembers` on every sync:

```go
// octo-server/modules/message/api_conversation.go:436
groupVailds, err = co.groupService.ExistMembers(groupNos, loginUID)
```

`ExistMembers` is implemented at `octo-server/modules/group/db.go:186-190` as:

```sql
SELECT group_no FROM group_member
  WHERE group_no IN (?) AND uid = ? AND is_deleted = 0
```

No `group_member` row → the channel is dropped from the user's session list, full stop. Even if WuKongIM has pushed messages from the channel to the user's client, the next sync will not surface the channel in the conversation list. There is no fallback path; `ExistMembers` is the sole gate.

Therefore the implementation is forced:

1. **Insertion point**: the message-delivery hook. When a user *first receives* a message from a `visibility = 1` channel, the delivery hook executes `INSERT IGNORE INTO group_member (uid, group_no, role=common, status=1, is_implicit=1)`.
2. **Bound on row growth**: a 1 000-member Space × 50 public channels × ~20% active-in-channel ratio ≈ 10 000 rows per Space. Single-table layout up to ~100 000 rows is fine; no partitioning required.
3. **`is_implicit` semantics**: a new column. Do **not** overload `status` or `role`. Both have well-defined existing semantics that other queries depend on; conflating them with implicit-subscription state would corrupt queries unrelated to this RFC.
4. **Cleanup paths — five of them**, listed exhaustively. Any one of them missed is an implicit-access leak:
   - `space_member.{remove, status 1→0}` → delete `is_implicit = 1` rows for that user across all `visibility = 1` channels of the Space.
   - `space.status 1 → 0` → delete *all* `is_implicit = 1` rows for that Space.
   - `group.visibility 1 → 0` (the wipe) → delete `is_implicit = 1` rows for that channel.
   - User-initiated "delete conversation" → soft-hide via a session-state column; do **not** delete the row.
   - `group.status = dissolved` → `is_deleted = 1` cascades through `group_member`; no special action.
5. **`/v1/groups/{no}/members` API**: must hide `is_implicit = 1` rows by default (a 1 000-member Space's full-roster page would be unusable otherwise); admin tools may opt in to including them.

**Effort**: Q6 alone — the lazy-insertion code, the five cleanup paths, and the test matrix that covers them — is roughly 4 person-days. The v0.1 draft estimated 4–5 days for *all of phase 1*; that estimate is unrealistic.

### Q7 — WuKongIM subscriber-cache consistency ("hidden depth")

The v0.1 draft proposed a 1-hour "spike" to verify whether WuKongIM caches subscriber lists at delivery time. **The spike is not necessary; the answer is settled by reading the source.**

| Path | Location | Caller |
|---|---|---|
| Delivery reads subscribers | `WuKongIM upstream — internal/channel/handler/event_distribute.go:337-359 (h *Handler).getSubscribers` | message delivery main flow |
| → delegates to | `event_distribute.go:338 service.Store.GetSubscribers` | ↓ |
| → delegates to | `pkg/cluster/store/channel.go:57-59 (s *Store).GetSubscribers` | ↓ |
| → reads from | `pkg/wkdb/subscriber.go:55 (wk *wukongDB).GetSubscribers` | pebble persistence |

Writes go through:

- `IMAddSubscriber` / `IMRemoveSubscriber` / `IMCreateOrUpdateChannel` HTTP API → WuKongIM internally translates into `Store.AddSubscriber / RemoveSubscriber / RemoveAllSubscriber` cluster commands → wkdb pebble write.

The `IDatasource.GetSubscribers` interface defined at WuKongIM upstream `internal/server/datasource.go:60` has **no callers** in the delivery path. A grep across upstream confirms this: the interface exists, the implementation in `octo-server/modules/webhook/api_datasource.go:84-113` is a fully-functional handler, but delivery never invokes it. (One administrative path — `internal/manager/manager_systemaccount.go:174` — does call out to the datasource, but that is for system-account lookups, not subscriber resolution.)

**Conclusion**: eager fanout is mandatory. There is no delivery-time fallback that would compensate for a missed `IMAddSubscriber` write.

#### Optional confirming probe (kept for written-record purposes only)

```
1. Enable WuKongIM's datasource webhook (set WK_DATASOURCE_ADDR to point at octo-server).
2. Add a Prometheus counter to octo-server/modules/webhook/api_datasource.go: getSubscribers handler:
     wk_datasource_getSubscribers_total{channel_type=...}
3. In a dev environment: create a regular group, send 100 messages.
4. Observe the counter:
     - 0 hits → confirms delivery does not call the datasource. Expected outcome.
     - >0 hits → identify which channel_type triggers it; this would change the analysis.
5. Concurrently observe wkdb subscriber-bucket size; confirms IMAddSubscriber writes through to pebble.
```

Expected result: counter = 0. The probe is for confirmation, not for decision-making.

---

## 3. Five blind spots not covered by the v0.1 §8 risk list

### Blind spot #1 (P0) — RFC core architecture relied on a wrong assumption

**Issue**. The v0.1 draft built §2 / §4 / §7 Q7 on the premise that *"reusing the thread feature's `IMDatasource.Subscribers` lazy-resolution"* would deliver public-channel subscriber routing for free. That premise does not hold in production.

**Evidence**.

- WuKongIM's delivery path reads from its own pebble-backed `wkdb` (call chain in §2 Q7 above).
- `octo-server`'s webhook datasource (`modules/webhook/api_datasource.go:84-113`) is a fully-implemented handler that is **not invoked on delivery**. It is reachable from the admin / system-account paths only.
- Even thread subscribers — which v0.1 cited as the "lazy" precedent — are written eagerly:
  - `octo-server/modules/group/service.go:1873-1905 addUsersToGroupThreads` writes `IMAddSubscriber` against every thread channel when a user joins the parent group.
  - `octo-server/modules/thread/service.go:1051+ RemoveUserFromGroupThreads` writes `IMRemoveSubscriber` when the user leaves.
  - `octo-server/modules/thread/service.go:617 IMCreateOrUpdateChannelInfo` passes the parent group's full member list when a thread is created.
- `octo-server/modules/thread/1module.go:113-134` *does* expose an `IMDatasource.Subscribers` callback, but it is consulted only by a handful of webhook administrative paths, never by delivery.

**Recommended change**.

`ResolveGroupSubscribers` is repositioned as a *local fanout-difference helper*, not a primary subscriber source. Every subscriber-affecting state change must explicitly call `IMAddSubscriber` / `IMRemoveSubscriber`. Five new event hooks:

| Event | Action |
|---|---|
| `group.visibility 0 → 1` | enumerate `space_member.status = 1`; subtract existing `group_member`; `IMAddSubscriber` the difference; write `channel_offset` lower bound |
| `group.visibility 1 → 0` | enumerate current implicit subscribers; `IMRemoveSubscriber`; delete `is_implicit = 1` rows |
| `space_member.add` | for every `visibility = 1` channel of the Space, `IMAddSubscriber` for the new uid |
| `space_member.remove` (incl. `status 1 → 0`) | symmetric removal |
| `space.status 1 → 0` | batch `IMRemoveSubscriber` across every `visibility = 1` channel of the Space |
| `group_member.add` (explicit invite) | already calls `IMAddSubscriber`; no change needed |

Phase-1 effort changes from 4–5 days (v0.1 estimate) to 8–10 days (eager-fanout estimate, including thread cascade for each hook). v0.2 reflects the new estimate.

---

### Blind spot #2 (P0) — `is_implicit` cleanup is not centralised

**Issue**. Q6's lazy `is_implicit` insertion has *five* cleanup triggers (listed in §2 Q6 above). The v0.1 draft does not enumerate them; without an exhaustive list, it is easy to ship four of the five and leave a residual leak.

**Why this is P0**. Every missing cleanup path is an implicit-access leak. Concrete failure modes:

- Path 1 missed → a user who left the Space continues to see the channel in their session list.
- Path 2 missed → after a Space is disabled, every former Space-member who was lazy-inserted into a public channel still has a `group_member` row; if the Space is ever re-enabled they re-appear as members of the (now-stale) public channel.
- Path 3 missed → after a `1 → 0` wipe, "wiped" users still see the channel in their session list because their `group_member` row survives.
- Path 4 confused with deletion → user-initiated "delete conversation" hard-deletes the row, and the next message re-creates it (the channel "comes back from the dead").
- Path 5 already handled by the existing `is_deleted = 1` cascade; the only requirement is to not break it.

**Recommended change**. Centralise in `cleanupImplicitMembers(spaceID, uid, scope)` and call it from every trigger. Each path gets a unit test:

| # | Trigger | Test |
|---|---|---|
| 1 | leave Space | assert no `is_implicit = 1` rows survive across the Space's public channels |
| 2 | disable Space | assert all `is_implicit = 1` rows under that Space are gone |
| 3 | flip `1 → 0` | assert only `is_implicit = 0` rows remain on the channel |
| 4 | "delete conversation" | assert the row persists; assert a separate hide-flag is set |
| 5 | dissolve channel | assert `is_deleted = 1` cascade still applies |

Cleanup must run **in the same database transaction as the trigger**. Asynchronous cleanup is forbidden because it creates a window during which the user can still receive messages they should have lost.

---

### Blind spot #3 (P1) — Bot ownership semantics in `visibility = 1` channels

**Issue**. The v0.1 draft does not say what happens to bots in a public channel. Existing OCTO semantics are clear for private channels: a bot's lifecycle follows its inviter (`octo-server/modules/group/db.go:702-728 QueryBotsInvitedByUIDTx`), and when an inviter leaves the channel, their bots cascade-leave with them. In a `visibility = 1` channel, the implicit-subscriber set has no obvious "inviter" relationship — every Space member is implicitly subscribed; nobody invited the bot via that path.

Open questions the v0.1 draft does not answer:

- Is a bot lazy-inserted into the `is_implicit` set when its owner is in the Space? (Should not be; bots are not Space-members.)
- Does the bot-ownership-validation path (`octo-server/modules/group/bot_ownership.go`) confuse "Space-implicit subscriber" for "channel-inviter"?
- After a `0 → 1 → 0` round-trip: which bots survive and which are evicted?

**Recommended change**. v0.2 explicitly states: bots are *never* lazy-fanned-out via the implicit-subscriber path. A bot enters a `visibility = 1` channel only via an explicit `group_member` row written by an explicit invitation flow. The fanout difference computation skips rows where `is_robot = 1`. Bot-lifecycle code is unchanged.

---

### Blind spot #4 (P1) — `Forbidden` (mute-all) is silent on Space admins

**Issue**. The current behaviour at `octo-server/modules/group/1module.go:83-94 Whitelist`:

```go
if groupInfo.Forbidden == 1 {
    return api.groupService.GetMemberUIDsOfManager(channelID)
}
```

`GetMemberUIDsOfManager` returns rows of `group_member` where `role = manager`. Space admins are not in this list unless they have been explicitly promoted to channel manager. In a `visibility = 1` channel with `Forbidden = 1`, a 1 000-member Space's admin **cannot post an announcement** — neither in the existing v0.1 design nor in the recommended v0.2 design.

**Two paths**.

- **(a)** Document the limitation: "during a `Forbidden` window on a public channel, a Space admin who needs to broadcast must first be promoted to channel manager". This keeps the channel-level permission model self-contained and avoids a hidden override.
- **(b)** Extend `Whitelist` to include Space admins automatically. This contradicts the §6 guarantee that *visibility-flip permissions = `creator + manager`, no Space-admin override*; the same logic that excludes Space admins from visibility flips should exclude them from speech-during-mute privileges.

**Recommendation: (a)**. v0.2 documents the restriction in §2.7 (Permission matrix) and the operations runbook; the `Whitelist` callback is unchanged.

---

### Blind spot #5 (P1) — Migration is missing `ALGORITHM` / `LOCK` clauses

**Issue**. v0.1 §3.1 specifies:

```sql
CREATE INDEX idx_group_space_visibility ON `group` (space_id, visibility, status);
ADD COLUMN visibility SMALLINT DEFAULT 0;
```

without `ALGORITHM` or `LOCK` clauses. On MySQL 8.0 the default behaviour will *usually* run as `INSTANT` for the column add and `INPLACE` for the index create — but not always. Specifically:

- If the `group` table uses `ROW_FORMAT = COMPRESSED`, `ADD COLUMN` falls back from `INSTANT` to `INPLACE` and rebuilds the table; metadata locks last for minutes on a 100 000-row table.
- If a long-running transaction is active when the migration acquires its metadata lock, the migration blocks all writes until the long transaction completes.
- `ROLL BACK`: a `DROP COLUMN` is *not* an `INSTANT` operation; if the migration ever needs to be reverted, the rollback requires a maintenance window.

**Quantitative estimate** at 100 000 rows on MySQL 8.0:

| Operation | Expected algorithm | Lock | Approximate duration |
|---|---|---|---|
| `ADD COLUMN visibility SMALLINT NOT NULL DEFAULT 0` | `INSTANT` | metadata-only, ms | < 100 ms |
| `CREATE INDEX idx_group_space_visibility (space_id, visibility, status)` | `INPLACE` | concurrent reads/writes allowed | 5–30 s |

**Recommended change**. v0.2 §4.1 specifies the migration as:

```sql
-- +migrate Up
ALTER TABLE `group`
  ADD COLUMN `visibility` SMALLINT NOT NULL DEFAULT 0
    COMMENT '0 = private, 1 = public_in_space',
  ALGORITHM=INSTANT, LOCK=NONE;

CREATE INDEX `idx_group_space_visibility`
  ON `group` (`space_id`, `visibility`, `status`)
  ALGORITHM=INPLACE, LOCK=NONE;

-- +migrate Down
DROP INDEX `idx_group_space_visibility` ON `group` ALGORITHM=INPLACE, LOCK=NONE;
ALTER TABLE `group` DROP COLUMN `visibility`;
```

A `pt-online-schema-change` dry-run against a staging copy of the production data is required before the production rollout.

The companion `group_visibility_change` table (§4.2 of v0.2) is a fresh `CREATE TABLE` with no migration risk.

---

## 4. §4.3 thread inheritance — is "zero extra work" really true?

**Verdict**: ✅ *for the subscriber-routing dimension only*; ⚠️ *for two side-effects the v0.1 draft glosses over*.

### Inheritance through the datasource callback

`octo-server/modules/thread/1module.go:113-134` defines the thread `IMDatasource.Subscribers` callback as `groupService.GetMembers(groupNo)`, which itself reads `group_member where is_deleted = 0`. So if the parent channel becomes public, *and* if the datasource callback were on the delivery path, the thread would automatically inherit the new subscribers.

But — as established in §5 — the datasource callback is *not* on the delivery path. The actual thread-subscriber writes happen at:

- `octo-server/modules/thread/service.go:617 IMCreateOrUpdateChannelInfo` — writes the parent group's full member list at thread creation.
- `octo-server/modules/group/service.go:1873-1905 addUsersToGroupThreads` — writes `IMAddSubscriber` for every thread when a new member joins the parent group.

### Recommended change

In the eager-fanout architecture (blind spot #1), every parent-channel `visibility 0 → 1` hook **must** additionally fan out to the parent's threads:

```
On group.visibility 0 → 1:
  1. UPDATE group SET visibility = 1
  2. newSubscribers = active space_members − existing group_members
  3. IMAddSubscriber(parent group, newSubscribers)
  4. SELECT every thread where parent = groupNo AND status != deleted
  5. for each thread: IMAddSubscriber(thread, newSubscribers)   ← extra step
  6. write channel_offset lower bound on parent and on each thread
```

Without step 5 a new Space member sees the parent channel's top-level messages but not its threaded replies — a subtle, easy-to-miss bug.

### Two side-effects the v0.1 draft does not call out

1. **`thread_member` semantics are preserved**. The `thread_member` table records *explicit* thread joins only; it is unaffected by `visibility = 1` (an implicit subscriber to the parent group does not become an implicit `thread_member`). `IsMember(uid)` checks against the explicit roster, which is correct.
2. **Thread "leave" becomes a UI fiction**. A user who hits "leave thread" in the client does not actually unsubscribe at the IM layer — `IMRemoveSubscriber` would fight the next delivery hook, which would re-add them. The v0.2 RFC documents this: *under a public parent, leaving a thread suppresses the thread in the UI but does not stop messages from being delivered*.

---

## 5. §7 Q7 — the WuKongIM subscriber-cache investigation

(Detailed evidence already presented in §2 Q7.)

The v0.1 draft proposed a 1-hour spike. The spike is not required; the static analysis already determines the answer. The recommended-but-optional 30-minute confirming probe (also in §2 Q7) exists only so the investigation has a written paper trail.

The structural conclusion: WuKongIM delivery reads its subscriber set from `wkdb` only; the `IDatasource.GetSubscribers` callback in `octo-server`'s webhook handler is an unused implementation of an unused interface on the hot path; eager `IMAddSubscriber` / `IMRemoveSubscriber` writes are the only mechanism for keeping `wkdb` correct.

---

## 6. Q2 historical-message visibility — the full implementation skeleton

**Decision**: reuse `channel_offset` (option (a) from §2 Q2 above; (b) and (c) rejected).

### Migration

Nothing new — the existing `channel_offset` table is sufficient.

### Visibility `0 → 1` switch handler

```go
func (s *Service) SwitchVisibilityToPublic(groupNo, operator string) error {
    tx, _ := s.ctx.DB().Begin()
    defer tx.RollbackUnlessCommitted()

    // 1. Lock the group row.
    var group Model
    err := tx.SelectBySql(
        "SELECT * FROM `group` WHERE group_no=? AND status=? FOR UPDATE",
        groupNo, GroupStatusNormal,
    ).LoadOne(&group)
    if err != nil {
        return err
    }
    if group.Visibility == GroupVisibilityPublicInSpace {
        return nil // idempotent
    }
    if group.SpaceID == "" {
        return errors.New("personal-space groups cannot be public")
    }

    // 2. Compute newSubscribers = active space_member − existing group_member.
    var newUIDs []string
    _, err = tx.SelectBySql(
        `SELECT sm.uid FROM space_member sm
           JOIN space s ON s.space_id=sm.space_id AND s.status=1
           WHERE sm.space_id=? AND sm.status=1
             AND sm.uid NOT IN (
               SELECT uid FROM group_member WHERE group_no=? AND is_deleted=0
             )`,
        group.SpaceID, groupNo,
    ).Load(&newUIDs)

    // 3. Snapshot the current message-seq as a lower bound.
    maxSeq, _ := s.messageDB.queryMaxMessageSeq(
        groupNo, common.ChannelTypeGroup.Uint8(),
    )

    // 4. Write channel_offset lower bound for every newly-implicit subscriber.
    for _, uid := range newUIDs {
        s.channelOffsetDB.insertOrUpdateTx(&channelOffsetModel{
            UID: uid, ChannelID: groupNo, ChannelType: 2, MessageSeq: maxSeq,
        }, tx)
    }

    // 5. Flip visibility.
    _, err = tx.UpdateBySql(
        "UPDATE `group` SET visibility=? WHERE group_no=?",
        GroupVisibilityPublicInSpace, groupNo,
    ).Exec()

    // 6. Audit row.
    _, err = tx.InsertInto("group_visibility_change").Values(...).Exec()

    if err := tx.Commit(); err != nil {
        return err
    }

    // 7. Post-commit IM write.
    if err := s.ctx.IMAddSubscriber(&config.SubscriberAddReq{
        ChannelID:   groupNo,
        ChannelType: common.ChannelTypeGroup.Uint8(),
        Subscribers: newUIDs,
    }); err != nil {
        s.Error("IMAddSubscriber failed after visibility flip", zap.Error(err))
        // ⚠️ DB committed but IM not reconciled — enqueue compensation job.
    }

    // 8. Cascade to threads.
    s.addUsersToGroupThreads(groupNo, newUIDs)

    // 9. System message announcement.
    s.ctx.SendCMD(...)
    return nil
}
```

### Reverse direction (`1 → 0` wipe)

Symmetric: enumerate `is_implicit = 1` rows, batch-delete them, batch `IMRemoveSubscriber` against the channel and every thread. See v0.2 §3.2.2 for the full sequence.

### Edge cases worth nailing in unit tests

- **`allow_view_history_msg = 1` interaction**. Without intervention, an existing channel with `allow_view_history_msg = 1` plus a fresh `visibility 0 → 1` flip would let new implicit subscribers read history (they're new joiners, and the channel "lets new joiners read history"). This violates Q2. v0.2 specifies: at visibility-flip time, `allow_view_history_msg` is treated as `0` regardless of its configured value, for the new implicit subscribers only.
- **Subsequent explicit invite of a previously-implicit user**. The standard invite path writes `channel_offset = 0` (or the configured lower bound) per the existing logic. This correctly overrides the visibility-flip lower bound, so a user who is later explicitly invited can read full history if `allow_view_history_msg = 1`. Both the invite and the visibility-flip paths must converge on this behaviour.

---

## 7. Q3 close-down — strong justification for "wipe to explicit roster"

(Already covered in §2 Q3.)

The v0.1 draft argued for wipe-to-explicit because the snapshot-and-freeze alternative was *"implementation-complex"*. That justification is weak; the strong justification is the **social contract**: implicit subscription is bound to *Space membership*, not to the channel. Reverting visibility must therefore unwind the implicit-subscription relationship, not promote it to explicit membership.

v0.2 §2.4 carries the strong justification verbatim and adds the "approximately N affected" UI confirmation requirement.

---

## 8. Migration timing for `idx_group_space_visibility`

(Already covered in blind spot #5.)

Summary: at 100 000 rows on MySQL 8.0 with default `ROW_FORMAT = DYNAMIC`, `ALGORITHM=INSTANT` for the column add (< 100 ms) and `ALGORITHM=INPLACE, LOCK=NONE` for the index create (5–30 s) are acceptable. The clauses must be explicit in the DDL. The RFC additionally requires a staging dry-run via `pt-online-schema-change`, and a documented rollback path that does *not* attempt `DROP COLUMN` online.

---

## 9. Security audit

### 9.1 Cross-Space subscriber leakage

**Verdict**: ✅ no leakage *given the immutability invariant*.

Evidence: the `visibility = 1` branch of `ResolveGroupSubscribers` joins `space_member` against the channel's *own* `space_id`. A Space-B member cannot show up in the subscriber set of a Space-A channel because the `WHERE sm.space_id = ?` filter rejects them.

⚠️ **Edge case**: if `group.space_id` could be mutated after the channel had been published, the flip would double-subscribe (old Space members continue receiving from `wkdb`; new Space members get added as implicit subscribers). The fix is to declare and enforce `group.space_id` as immutable. A grep across `octo-server/modules/group/` finds no current code path that mutates `group.space_id`, so the invariant is declarative rather than constraint-enforced. v0.2 §4.5 documents this invariant.

### 9.2 XSS / injection on channel name, topic, notice

**Verdict**: ⚠️ same handling as today, but blast radius scales with Space size.

Evidence:

- `octo-server/modules/group/api_setting_action.go` writes channel name to the database without HTML escaping.
- `octo-server/modules/group/api.go newChannelRespWithGroupResp:170-235` returns the name unescaped, deferring to client-side rendering.
- Client-side escape is observed in the web client today, but trusting client-side escape across web / iOS / Android is fragile — a single client missing the escape becomes the path of least resistance.

In a `visibility = 0` channel, a script injection in the channel name renders for the (small) explicit roster. In a `visibility = 1` channel, the same payload renders for **every Space member**.

**Recommended change** (mandatory for v0.2 to ship):

1. Server-side `html.EscapeString` on `name`, `topic`, `notice` writes — at the API handler, before the database write. Front-end escaping remains as defence in depth.
2. `name VARCHAR(50)`, `topic VARCHAR(100)`, `notice VARCHAR(1000)` length caps enforced server-side.
3. A retroactive escape pass during the rollout window covers existing rows — channels that are about to become public should have their existing strings sanitised before they are exposed to a wider audience.

The same hardening also benefits private channels but is out of scope; v0.2 documents this as a future tightening.

### 9.3 Space-switch cross-pollution

**Verdict**: ✅ contained, given existing controls.

`octo-server/modules/message/api_conversation.go:295,436` runs `ExistMembers` on every sync; `octo-server/pkg/space/middleware.go` validates the `X-Space-ID` header against the requesting user's actual Space membership. A user with cached Space-A channels in their client cannot have them surface when the UI switches to Space B.

⚠️ One known carry-over: a user who is simultaneously a `space_member` of Space A and an explicit `group_member` (e.g. external collaborator) on a Space-B channel will continue to see that Space-B channel under Space-B view. This is the existing external-group behaviour, not a new leak. v0.2 documents it explicitly so the behaviour is not later mistaken for a regression.

### 9.4 SQL injection

**Verdict**: ✅ safe.

Evidence: every query in §3 binds dynamic values through `?` placeholders. No string interpolation. Identifiers like `spaceID` and `groupNo` come from authenticated session state, not from request bodies.

The one defensive-depth requirement: validate `groupNo` against `^[0-9a-f]{32}$` at the API boundary so that a malformed input is rejected before it reaches the database driver. This is a cheap belt-and-braces check that v0.2 specifies as part of the API-handler contract.

### 9.5 Blacklist enforcement

**Verdict**: ⚠️ requires explicit handling on the `visibility = 1` path.

The current `Blacklist` callback (`octo-server/modules/group/1module.go:80-82`) returns rows from `group_member` where `status = blacklist`. In a `visibility = 1` channel a blacklisted user does not necessarily have a `group_member` row at all (they are an implicit subscriber). Therefore the manager's blacklist action must explicitly write a `group_member` row:

```
manager blacklists uid X on a public channel:
  INSERT INTO group_member (uid, group_no, role=common, status=blacklist, is_implicit=1)
  ON DUPLICATE KEY UPDATE status=blacklist
  IMRemoveSubscriber(groupNo, [X])
  IMRemoveSubscriber(every thread under groupNo, [X])
```

The implicit-subscriber resolver must then `EXCEPT` blacklisted rows from the lazy-fanout candidate set; without this filter, the next message delivery would attempt to re-fanout to the blacklisted user.

v0.2 §5.4 formalises this. The alternative — a separate `group_blacklist` table — was considered and rejected in favour of reusing the existing `status = blacklist` semantics.

---

## 10. Verdict

**Recommendation**: do not implement v0.1 as drafted. Adopt v0.2 (the companion document) which incorporates the eight recommended changes summarised in §11.

### Three strongest reasons against shipping v0.1

1. **The architecture rests on a wrong assumption** (blind spot #1). "Reuse `IDatasource.GetSubscribers`" does not work in production: WuKongIM delivery reads from `wkdb`, not from the datasource callback. Eager fanout is mandatory. Effort scales by ≈ 2× and five new event hooks must be added. §2 / §4 / §7 Q7 of the v0.1 draft all need to be rewritten.
2. **The session-list path is a P0 blocker** (blind spot #2 + Q6). `ExistMembers` filters out channels for which the user has no `group_member` row. Lazy `is_implicit` insertion is not an optimisation — it is required for the channel to appear in the session list at all. Cleanup spans five distinct trigger paths; missing any one leaks implicit access. v0.1's Q6 marks lazy-insertion as a *preference*; it must be promoted to *the only viable design*, with an exhaustive cleanup-path matrix and unit tests for each.
3. **The effort estimate is ~2× too low**. v0.1 budgets 4–5 days for "Migration + ResolveGroupSubscribers + datasource hookup + visibility API + unit tests". The realistic scope is 8–12 days for "5 event hooks + 6 fanout sites + lazy `is_implicit` + 5 cleanup paths + thread cascade + Q2 lower bound". Total v0.2 estimate including UI and rollout is ~21 person-days, against v0.1's ~11.5.

### Three strongest reasons in favour of the *direction*

1. **The product motivation is sound**. A per-Space public-channel concept is table stakes for any workplace IM at the scale OCTO is targeting. Comparable products all ship it. The current "everything is private" model creates real onboarding friction that scales with Space size.
2. **The data-model choices are right**. Staying on `ChannelTypeGroup(2)` rather than switching to `ChannelTypeCommunity(4)` avoids a 2–4-month full-stack overhaul. The single new column on `group` plus an index is minimally intrusive. Orthogonality to `is_external_group` keeps the matrix clean.
3. **Once the architecture is corrected, the structural risk is low**. The eager-fanout pattern that v0.2 mandates is identical to the pattern OCTO's thread feature has been running in production for the past six months. The remaining risks (subscriber-fanout latency, session-list churn, Space-admin governance) are operational rather than architectural and are addressable through the rollout plan in v0.2 §6.

---

## 11. Recommended changes against v0.1 (the eight patches v0.2 incorporates)

### Recommended change 1 — §2 decision table: subscription-mode row rewrite

```diff
 | Subscription model |
- | Reuse the thread feature's verified IMDatasource.Subscribers lazy resolution: when visibility = 1, the callback returns active space_member; when visibility = 0, it returns group_member (status quo). | Thread has run this in production for six months; the pattern is reliable. |
+ | Eager fanout (same shape as the thread feature): on visibility flips, on Space-membership changes, on channel-membership changes, explicitly call IMAddSubscriber / IMRemoveSubscriber to write into WuKongIM's wkdb. | The delivery path reads from wkdb only (`event_distribute.go:338 service.Store.GetSubscribers`); the IDatasource callback is not exercised on delivery. The thread feature in production already follows the eager pattern (`addUsersToGroupThreads` / `RemoveUserFromGroupThreads`). |
```

### Recommended change 2 — §4 reframed as eager fanout

```diff
- ### 4.1 New: pkg/space/group_subscribers.go
- ResolveGroupSubscribers returns a channel's subscriber list given its visibility.
- Used by IMDatasource.Subscribers and thread.Subscribers callbacks.
+ ### 4.1 New: pkg/space/group_subscribers.go (a *local* helper, NOT a datasource source)
+ ResolveGroupSubscribers computes the subscriber set the channel should have under its current visibility.
+ Used as the difference computer for the event hooks below; does NOT replace the eager IMAddSubscriber / IMRemoveSubscriber writes on the hot path.

  ### 4.2 modules/group/1module.go
- IMDatasource.Subscribers currently returns the group_member list. Change it to delegate to ResolveGroupSubscribers.
+ Leave IMDatasource.Subscribers unchanged (the existing webhook fallback continues to work). Add the following event hooks; each one calls ResolveGroupSubscribers for the difference, then issues IMAddSubscriber / IMRemoveSubscriber:
+ - SwitchVisibilityToPublic(groupNo)   (0 → 1)
+ - SwitchVisibilityToPrivate(groupNo)  (1 → 0, the wipe)
+ - OnSpaceMemberAdd(spaceID, uid)
+ - OnSpaceMemberRemove(spaceID, uid)
+ - OnSpaceDisable(spaceID)

  ### 4.3 modules/thread/1module.go
- thread.Subscribers currently returns groupService.GetMembers(groupNo). Change it to delegate to ResolveGroupSubscribers — threads inherit parent visibility automatically.
+ Leave thread.Subscribers unchanged (also retained for the webhook fallback). Each parent-channel event hook must additionally enumerate non-deleted threads and issue IMAddSubscriber / IMRemoveSubscriber for each, mirroring the existing `octo-server/modules/group/service.go:1873-1905 addUsersToGroupThreads`.
```

### Recommended change 3 — §7 Q2 with a concrete mechanism

```diff
  ### Q2. private → public history visibility for new implicit subscribers
  Recommendation: invisible. The transition is a clear product event.
- Reuse the group.allow_view_history_msg = 0 semantics for the new implicit subscribers. The IDatasource layer must distinguish "messages from before the flip" as a per-user lower bound — exact mechanism TBD by engineering review.
+ Implementation: reuse channel_offset (existing write path at `octo-server/modules/message/api.go:1855 insertOrUpdate`). At visibility 0 → 1, in the same transaction, write one row per (active_space_member − existing_group_member) uid: channel_offset(uid, group_no, channel_type=2, message_seq=current_maxSeq). On the lazy join (Q6), the new subscriber's session sync starts from the channel_offset lower bound and never sees pre-flip messages. Strong-form rule: at visibility flip, group.allow_view_history_msg is treated as 0 for the newly-implicit subscribers regardless of its configured value.
```

### Recommended change 4 — §7 Q3 with the strong (social-contract) justification

```diff
  ### Q3. public → private close
  Recommendation (b): wipe to the explicit roster. Keep creator + manager + explicitly-invited members; everyone else has to be re-invited.
- Reasoning: option (a) snapshot-and-freeze is implementation-complex and the UI is hard to express; option (c) one-way irreversibility is a poor product experience.
+ Reasoning: implicit subscription is bound to "I am in this Space → I see this channel", NOT to "I am a named member of this group". On close, promoting a thousand Space users to explicit group_member rows violates the original mental model and produces "I never joined this channel; why can't I leave it?" complaints. (a) is wrong, not "complex". (c) closes off recovery from a misconfigured 0 → 1 flip.
- Carry-over: keep a flip-history table (`group_visibility_change`?) for audit — TBD.
+ Required: the close-handler writes a `group_visibility_change` audit row including the affected uid list (schema in companion review §8). The operator UI must show a strong confirmation ("This will revoke channel access for approximately N Space members") before committing.
```

### Recommended change 5 — §7 Q6 promoted from "preference" to "the only design"

```diff
  ### Q6. Conversation state (last_read / mute / stick / unread)
- This is the largest implementation complexity for the implicit-subscription model. All of these states currently depend on a group_member row existing. Two paths:
- - (a) Lazy row insertion …
- - (b) New table group_member_implicit …
- - Preference: (a), with an is_implicit flag — minimal change. To be confirmed by engineering review.
+ (a) Lazy group_member(is_implicit = 1) is the only viable design, not a preference. Reasoning: session sync (`octo-server/modules/message/api_conversation.go:436`) hard-filters via `groupService.ExistMembers`. No group_member row → the channel is dropped from the user's session list. WuKongIM may have already pushed messages, but the sync API will not surface the channel without a row.
+
+ Insertion point: the message-delivery hook. On a user's first message receipt from a visibility = 1 channel: INSERT IGNORE INTO group_member (uid, group_no, role=common, status=1, is_implicit=1).
+
+ Five cleanup paths (any one missed = implicit-access leak):
+ 1. space_member.{remove, status 1→0} → delete is_implicit=1 rows for the user across the Space's visibility=1 channels; IMRemoveSubscriber.
+ 2. space.status 1 → 0 → batch-delete is_implicit=1 rows across all channels of the Space.
+ 3. group.visibility 1 → 0 (wipe) → delete is_implicit=1 rows; IMRemoveSubscriber.
+ 4. user "delete conversation" → set a session-state hide flag; do NOT delete the row.
+ 5. group dissolved (status=2) → existing is_deleted=1 cascade is sufficient.
+
+ API impact: GET /v1/groups/{no}/members hides is_implicit=1 by default (admin tools may opt in to including them).
+
+ Estimated row growth: 1 000-member Space × 50 public channels × ~20% active ≈ 10 000 group_member rows / Space. Single-table layout up to ~100 000 rows is fine; no partitioning required.
```

### Recommended change 6 — §7 Q7 rewritten with the static-analysis verdict

```diff
  ### Q7. WuKongIM subscriber-cache consistency (hidden depth)
- WuKongIM caches ChannelTypeGroup subscribers internally (the IMCreateOrUpdateChannelInfo path). Public channels must confirm the cache is invalidated on Space-membership changes; otherwise new Space joiners do not receive messages.
- - Required: a 1-hour spike to verify the WuKongIM subscriber-resolution path in production.
- - Contingency: if WuKongIM caches aggressively, push a manual cache-invalidation API call from octo-server on every space_member change.
+ Resolved without a spike. Static analysis of the WuKongIM upstream `develop` branch shows the delivery path reads from wkdb (`internal/channel/handler/event_distribute.go:338 service.Store.GetSubscribers` → `pkg/cluster/store/channel.go:57` → `pkg/wkdb/subscriber.go:55`). The `IDatasource.GetSubscribers` callback is not invoked on delivery (zero call sites in the upstream tree). All subscriber state must be written eagerly via IMAddSubscriber / IMRemoveSubscriber. WuKongIM's in-memory tag cache (`event_distribute.go:82,234 GetChannelTag / RemoveTag`) is invalidated internally by IMAddSubscriber; octo-server does not have to push an explicit invalidation.
```

### Recommended change 7 — §8 risk list extended

```diff
  ## 8. Identified risks
  1. Subscriber fanout explosion …
  …
  6. Migration safety …
+ 7. The original architecture rested on a wrong assumption (P0). v0.1 assumed the `octo-server`-side IDatasource.GetSubscribers webhook would resolve subscribers on delivery. See companion review blind spot #1.
+ 8. is_implicit cleanup spans five paths (P0). Any missed path is an implicit-access leak. See companion review blind spot #2 + §7 Q6.
+ 9. Bot ownership conflicts with implicit Space-membership (P1). Are bots lazy-fanned-out? Does the bot-inviter cascade run in public channels? See companion review blind spot #3.
+ 10. Forbidden mutes Space admins (P1). The whitelist returns role=manager only; Space admins are not auto-elevated. See companion review blind spot #4.
+ 11. Channel-name XSS blast radius scales with Space size (P1). Private-channel name XSS hits the explicit roster; public-channel name XSS hits every Space member. See companion review §9.2.
```

### Recommended change 8 — §9 phase plan re-estimated

```diff
  ## 9. Rollout plan
  | Phase | Scope | Effort |
  |---|---|---|
- | 0 | Spike: verify the WuKongIM subscriber-cache resolution path (1 hour) | 0.5 d |
- | 1 | Migration + ResolveGroupSubscribers + datasource hookup + visibility API + unit tests | 4–5 d |
- | 2 | Web channel-settings UI + public badge + session-list rendering | 3 d |
- | 3 | iOS / Android UI (no logic changes) | 2 d |
- | 4 | Gated rollout + fanout load test | 2 d |
+ | 0 | (No spike required — answered by static analysis; see companion review §5) | 0 d |
+ | 1a | Migration (incl. audit table) + model fields + visibility API + unit tests | 2 d |
+ | 1b | The five event hooks (visibility flip × 2 / space-member × 2 / space-disable) + thread cascade + compensation queue | 4 d |
+ | 1c | Lazy is_implicit insertion + the five cleanup paths + Q2 channel_offset lower bound + blacklist adaptation | 4 d |
+ | 1d | Integration tests (full Space + flip + leave + blacklist + forbidden + thread); 1k-member synthetic load | 2 d |
+ | 2 | Web channel-settings UI + public badge + session-list rendering (incl. two-tab consideration deferred to v2) | 4 d |
+ | 3 | iOS / Android UI (badge only, no logic) | 2 d |
+ | 4 | Single-Space rollout + fanout latency P99 monitoring + wkdb size monitoring + ramp to all production Spaces | 3 d |
+
+ Total: ~21 person-days, against v0.1's ~11.5.
```

---

## Closing note

The most useful single insight from this review: the original architectural premise — *"reuse the thread feature's `IMDatasource` callback"* — was a *partial* truth. The thread feature does have such a callback, but it is not what makes thread subscribers work in production. What actually keeps thread subscribers correct is eager `IMAddSubscriber` / `IMRemoveSubscriber` writes on every membership-changing event. The same pattern, applied to channels, is what makes public-in-Space channels viable.

v0.2 incorporates all eight recommended changes. After the recommended changes, the structural risk profile shifts from "is the architecture viable?" (open question, no) to "have we covered all five cleanup paths and all five event hooks?" (a finite, testable contract). The rollout plan in v0.2 §6 is built around answering the latter rather than the former.

This report is preserved alongside `0001-public-channels.md` as the design provenance for the structural choices in v0.2.
