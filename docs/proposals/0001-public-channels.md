# RFC-0001: Public-in-Space Channels

- **Status**: Accepted (after engineering review on 2026-05-14)
- **Author(s)**: Octo team
- **Created**: 2026-05-14
- **Originating discussion**: <https://github.com/Mininglamp-OSS/community/discussions/1>
- **Tracking issue**: <https://github.com/Mininglamp-OSS/octo-server/issues/23>
- **Implementation repos**: `octo-server`, `octo-lib`, `octo-web`

---

## 1. Motivation

OCTO's current group model is **private-by-default-and-only**: every channel requires explicit invitation, and the only way for a Space user to discover or join an internal discussion is for an existing member to add them. This is a deliberate carry-over from a 1:1 / small-team chat lineage, but it creates two concrete user-facing problems as Spaces grow past a handful of members:

1. **Onboarding friction.** A new joiner of a 200-person Space sees an empty conversation list. They cannot tell *what* internal channels exist (`#general`, `#engineering`, `#announcements`), let alone discover the right ones to join. Every joiner becomes an organic invitation request to whoever happens to be online — a tax that scales linearly with Space size.
2. **Announcement fragmentation.** Operators who want to broadcast to "everyone in the Space" today have no equivalent of a public channel. They fall back to scripts that bulk-add the entire member list to a private group, which (a) bloats `group_member` rows, (b) makes future additions a manual chore, and (c) gives leavers no way to "unsubscribe" without being individually kicked.

The standard solution shipped by every comparable workplace IM is **a per-Space concept of a public channel**: a channel whose membership is implicitly the Space, available to discover, browse, and post to without explicit invitation. Discord exposes this as the default server channel; Slack as the channel browser + `#general`; Feishu / Lark and WeCom as 频道 / 公告群. Across products the user model is unsurprising: *"if you are in this workspace, you can see this channel."*

This RFC introduces that capability to OCTO under the name **public-in-Space channels**: a single boolean (`group.visibility`) that promotes an existing private channel into a Space-public one without changing its underlying channel type, ID, message history, or thread structure. The key non-goal is *cross*-Space publication — that is a much larger directory / social-graph problem and is explicitly out of scope for v1 (see §2 *Out of scope*).

The remainder of this RFC specifies user-facing semantics (§2), the eager-fanout architecture that makes the feature work end-to-end on top of the existing WuKongIM delivery path (§3), the data model and migrations (§4), the security boundaries that distinguish a public channel from a leaked one (§5), and the rollout plan (§6). Open questions that explicitly do *not* block implementation are collected in §7.

---

## 2. User-facing semantics

### 2.1 The core concept

Each channel grows a single new attribute, `visibility`, with two possible values:

| `visibility` | Name | Effective subscriber set |
|---|---|---|
| `0` (default) | `private` | The explicit `group_member` rows, exactly as today. |
| `1` | `public_in_space` | Every active `space_member` of the channel's owning Space, plus any explicit `group_member` rows (e.g. external members invited individually). |

`visibility = 0` is the migration default. Every existing channel is unaffected by the rollout — this is a strictly additive feature.

### 2.2 Who can flip the switch

| Action | Permitted operators |
|---|---|
| Set `visibility 0 → 1` (open the channel to the Space) | Channel `creator` **or** channel `manager` |
| Set `visibility 1 → 0` (close the channel back to private) | Channel `creator` **or** channel `manager` |

Both operators **must be internal members** (`is_external = 0`). External members can never become channel managers (existing rule, enforced at `octo-server/modules/group/db.go:107-114`), which means they can never flip visibility — a deliberate property, not a workaround.

Space admins are *not* automatically granted visibility-flip rights. The reasoning: a public channel is a social commitment by the channel's owner, not a Space-wide property to be set by a Space admin. Allowing a Space admin to unilaterally publicise someone else's private channel would be a recurring source of user complaints. We can revisit this after six months of production data (§7).

Every visibility transition writes an immutable audit row to `group_visibility_change` (§4) recording operator, from-value, to-value, timestamp, and (for `1 → 0`) the list of users whose access was revoked.

### 2.3 Historical messages on `private → public`

When a channel transitions from `private` to `public_in_space`:

- Existing `group_member` rows continue to see the full history (no change for them).
- Newly-implicit subscribers (the Space members who become visible to the channel as a result of the flip) see **only messages sent at or after the flip**. They cannot scroll back into history.

This matches the user mental model: *"I joined the channel when it became public; I shouldn't be able to read what was said when it was a private discussion."* It is implemented by reusing OCTO's existing `channel_offset` mechanism (the same one that powers `allow_view_history_msg = 0` for private invites) — see §3.3 and §4.

When `visibility` is flipped back to `0`, the original `group_member` rows recover their full-history view automatically (their `channel_offset` was never written).

### 2.4 Closing back to private (`public → private`)

When `visibility` flips from `1` back to `0`, the channel is **wiped clean** — only the explicit roster (creator, managers, anyone explicitly invited via `group_member`) remains. Implicit subscribers are removed from the WuKongIM subscriber list and from the channel's session-list membership.

We considered three alternatives and reject the other two:

- **(a) Snapshot-and-freeze**: convert every implicit subscriber into an explicit `group_member` row at the moment of flip. Rejected because the user mental model when subscribing was *"I'm in this Space, so I see this channel"*, **not** *"I am a named member of this group"*. Promoting them to explicit members on close would generate "I never joined this channel, why can't I leave?" complaints.
- **(b) Wipe to the explicit roster** *(chosen)*. Aligns with the social contract: a private channel has always been an explicit-invitation construct; reverting visibility restores that property.
- **(c) Irreversible one-way switch**. Rejected because a misconfigured `0 → 1` flip would be unrecoverable. Reversibility is a safety property even if rare in practice.

The UI of the operator who triggers `1 → 0` must show a strong confirmation — *"This will revoke channel access for approximately N Space members"* — to make accidents loud.

### 2.5 Leaving the Space

When a user leaves a Space (or is removed, or the Space is disabled), every public channel of that Space automatically becomes invisible to them: they are unsubscribed from message delivery and the channels disappear from their session list. This is implemented by event hooks on `space_member.remove` and `space.status` (see §3.2).

Existing `group_member` rows (e.g. someone explicitly invited as an external) are *not* affected by Space membership changes — they continue to follow the existing private-membership rules.

### 2.6 Interaction with external groups

The new `visibility` axis is **orthogonal** to the existing `is_external_group` axis. All four combinations are valid:

| `visibility` | `is_external_group` | Meaning |
|---|---|---|
| `0` | `0` | Standard internal private channel (status quo). |
| `0` | `1` | Standard external group with explicitly-invited external members. |
| `1` | `0` | Internal channel published to the entire Space. |
| `1` | `1` | A public channel that *also* has external members on the explicit roster. |

The last row deserves attention: in a `visibility=1, is_external_group=1` channel, internal Space members are subscribed implicitly *and* explicit external members continue to participate. Both groups see each other. The channel name in the UI must be double-tagged ("公开 · 含外部成员" / "Public · External members") so operators don't accidentally publish discussions intended only for internal eyes.

### 2.7 Permission matrix

| Action | `visibility = 0` | `visibility = 1` |
|---|---|---|
| Send messages | `group_member` rows only | Implicit Space members + `group_member` rows; subject to `Forbidden` (see below) |
| View messages | `group_member` rows only | Same as above |
| Add explicit member | Channel `creator` / `manager` | Channel `creator` / `manager` |
| Remove explicit member | Channel `creator` / `manager` | Channel `creator` / `manager` (only affects explicit rows) |
| Blacklist | `manager` | `manager` (writes an explicit `group_member` row with `status = blacklist, is_implicit = 1`; see §5) |
| Forbidden (mute-all) | Only `manager` may post | Only `manager` may post — Space admins are **not** auto-elevated, by design |
| Set channel notice / topic | `manager` | `manager` |
| Flip `visibility` | `creator` or `manager` (internal only) | `creator` or `manager` (internal only) |
| Dissolve channel | `creator` | `creator` |

When `Forbidden = 1` on a `visibility = 1` channel, only channel managers can post. Space admins are not silently added to the post-allowlist — if a Space admin needs to broadcast during a `Forbidden` window, they must first be promoted to `manager` on the channel. This keeps the channel-level permission model self-contained and avoids creating a hidden Space-admin override path that would surface in audit reviews.

### 2.8 Out of scope (v1)

The following are deliberately *not* shipped in this RFC:

- **Cross-Space public channels.** A channel published to multiple Spaces or to the global member directory. Significant directory + social-graph work; requires a separate proposal.
- **Channel browser / discovery directory.** A "list every public channel I haven't joined" UI. The current implementation simply makes public channels appear in the regular session list of every Space member, on first message receipt.
- **One-click follow / unfollow.** A user-facing "follow channel" toggle distinct from explicit join. The implicit subscription is full-duty: there is no per-user follow flag in v1.
- **Q&A / poll mode.** Channel-level message-shape constraints (read-only announcements, threaded Q&A, polls). Out of scope.

---

## 3. Architecture

### 3.1 Why eager fanout, not lazy datasource

The naive design — and the path the v0.1 draft of this RFC initially proposed — is to lean on WuKongIM's `IDatasource.GetSubscribers` callback so that the subscriber set is computed lazily at message-delivery time. That intuition turns out to be wrong, in a way that is worth documenting because it shapes every other decision in this section.

The actual WuKongIM delivery path on the upstream `develop` branch is:

```
event_distribute.go:337  (h *Handler).getSubscribers
  └── service.Store.GetSubscribers          ← reads pebble-backed wkdb
      └── pkg/cluster/store/channel.go:57   (s *Store).GetSubscribers
          └── pkg/wkdb/subscriber.go:55     (wk *wukongDB).GetSubscribers
```

The delivery hot-path reads its subscriber list from WuKongIM's *own* pebble store (`wkdb`). The `IDatasource.GetSubscribers` callback that OCTO exposes via the webhook datasource (`octo-server/modules/webhook/api_datasource.go:84-113`) **is not invoked** on the delivery hot-path — a `grep` of `Datasource.GetSubscribers` across the upstream WuKongIM tree shows zero call sites; the interface is defined but never dereferenced for subscriber resolution. (It is invoked for system-account lookups, which is a separate path.)

This mirrors how OCTO's existing thread (子区) feature works in production: thread subscribers are written eagerly, not resolved lazily. When a user joins the parent group, `octo-server/modules/group/service.go:1873-1905 addUsersToGroupThreads` calls `IMAddSubscriber` against every thread channel; when a user leaves, `octo-server/modules/thread/service.go:1051+ RemoveUserFromGroupThreads` calls `IMRemoveSubscriber`. The thread's `IDatasource.Subscribers` callback (`octo-server/modules/thread/1module.go:113-134`) exists but is only consulted by a few rarely-hit webhook administrative paths, not by message delivery.

The conclusion that drives the rest of this section: **public-in-Space channels must follow the same eager-write pattern**. Every Space-membership and visibility transition must explicitly call `IMAddSubscriber` / `IMRemoveSubscriber` so that WuKongIM's wkdb has the correct subscriber set in advance of the next delivery.

The local helper function `pkg/space/group_subscribers.go: ResolveGroupSubscribers(groupNo) → []uid` is therefore a *fanout-difference computer*, not a delivery-time data source. It is consulted by the event hooks below to figure out *what to write*, and it remains wired into the existing webhook datasource callbacks as a fallback for the few administrative paths that do invoke them — but it is not the primary subscriber source.

### 3.2 The five event hooks

Five new event hooks own the entire subscriber-list maintenance contract for `visibility = 1` channels. Each hook runs in its own database transaction; the IM write happens *after* commit, with a compensation queue covering the (rare) commit-then-IM-fail case.

#### 3.2.1 `SwitchVisibilityToPublic(groupNo)` — `visibility 0 → 1`

```
1. SELECT ... FOR UPDATE on the group row; assert visibility == 0 and space_id != ""
2. Compute newSubscribers =
     active space_member of group.space_id
     MINUS existing group_member of groupNo (any role, is_deleted=0)
3. For each uid in newSubscribers:
     INSERT INTO channel_offset(uid, group_no, channel_type=2, message_seq=current_maxSeq)
     ON DUPLICATE KEY UPDATE message_seq = current_maxSeq
4. UPDATE `group` SET visibility = 1 WHERE group_no = ?
5. INSERT INTO group_visibility_change (operator, from=0, to=1, timestamp)
6. COMMIT
7. (post-commit) IMAddSubscriber(channelID=groupNo, channelType=2, subscribers=newSubscribers)
8. (post-commit) For each non-deleted thread under groupNo:
     IMAddSubscriber(channelID=threadChannelID, channelType=<thread>, subscribers=newSubscribers)
9. (post-commit) Send a system message announcing the visibility change.
```

Step 3 is the mechanism by which historical messages stay invisible to the new implicit subscribers (§2.3). The implementation pattern is identical to the existing `octo-server/modules/message/event.go:413-477 updateMembersChannelOffset`, which already writes `channel_offset` rows for new joiners of channels with `allow_view_history_msg = 0`.

Step 8 is critical and easy to forget. Threads are independent IM channels with their own subscriber lists in wkdb; a public-channel flip that did not also fan out to threads would leave new Space members able to see top-level messages but *not* thread replies. Test coverage for thread fanout on visibility flip is mandatory.

#### 3.2.2 `SwitchVisibilityToPrivate(groupNo)` — `visibility 1 → 0` (the wipe)

```
1. SELECT ... FOR UPDATE on the group row; assert visibility == 1
2. Compute removedSubscribers =
     current implicit subscribers (group_member rows where is_implicit = 1, is_deleted = 0)
3. UPDATE group_member SET is_deleted = 1 WHERE group_no = ? AND is_implicit = 1
4. UPDATE `group` SET visibility = 0 WHERE group_no = ?
5. INSERT INTO group_visibility_change (operator, from=1, to=0, affected_uids_json=removedSubscribers, timestamp)
6. COMMIT
7. (post-commit) IMRemoveSubscriber(channelID=groupNo, channelType=2, subscribers=removedSubscribers)
8. (post-commit) For each non-deleted thread under groupNo:
     IMRemoveSubscriber(channelID=threadChannelID, channelType=<thread>, subscribers=removedSubscribers)
```

Step 3 is the *unique* place where `is_implicit` rows are reaped during a visibility flip. The five global cleanup paths for `is_implicit` rows are listed in §4.3; any cleanup-path bug here means revoked users continue to receive messages or appear in `ExistMembers` results.

#### 3.2.3 `OnSpaceMemberAdd(spaceID, uid)`

Triggered when a user is added to a Space (status flipping from `0 → 1` is treated identically to a new row).

```
1. SELECT group_no FROM `group`
   WHERE space_id = ? AND visibility = 1 AND status = active
2. For each public channel:
     - INSERT IGNORE channel_offset(uid, group_no, channel_type=2, message_seq=current_maxSeq)
       (so that this user, like any other newly-implicit subscriber, does not see history)
     - IMAddSubscriber(channelID=groupNo, channelType=2, subscribers=[uid])
     - For each non-deleted thread: IMAddSubscriber(threadChannelID, ..., [uid])
```

The `is_implicit` `group_member` row is **not** written here — it is written lazily on first message receipt (see §3.3 and §4.3).

#### 3.2.4 `OnSpaceMemberRemove(spaceID, uid)` (includes `space_member.status 1 → 0`)

```
1. SELECT group_no FROM `group`
   WHERE space_id = ? AND visibility = 1 AND status = active
2. For each public channel:
     - DELETE FROM group_member WHERE group_no = ? AND uid = ? AND is_implicit = 1
     - IMRemoveSubscriber(channelID=groupNo, channelType=2, subscribers=[uid])
     - For each non-deleted thread: IMRemoveSubscriber(threadChannelID, ..., [uid])
```

#### 3.2.5 `OnSpaceDisable(spaceID)` — `space.status 1 → 0`

The whole-Space teardown.

```
1. SELECT group_no FROM `group` WHERE space_id = ? AND visibility = 1
2. For each public channel:
     - DELETE FROM group_member WHERE group_no = ? AND is_implicit = 1
     - SELECT every uid that was implicit on this channel
     - IMRemoveSubscriber(channelID=groupNo, channelType=2, subscribers=batch)
     - For each non-deleted thread: IMRemoveSubscriber(threadChannelID, ..., batch)
```

### 3.3 Lazy `is_implicit` insertion on first message receipt

Session-list visibility (the user's "channel list") is gated by `octo-server/modules/message/api_conversation.go:436`, which calls `groupService.ExistMembers(groupNos, loginUID)`; the underlying SQL at `octo-server/modules/group/db.go:186-190` reads:

```sql
SELECT group_no FROM group_member
  WHERE group_no IN (?) AND uid = ? AND is_deleted = 0
```

If the user has *no* `group_member` row for a channel, the channel is filtered out of their session list — even if WuKongIM has already pushed messages from it. Eager IM-subscription alone is therefore not enough: every implicit subscriber must also have a `group_member` row for the session-list path to surface the channel.

We could pre-write the row in `OnSpaceMemberAdd` and `SwitchVisibilityToPublic`, but for a 1000-member Space with 50 public channels that is 50 000 rows written upfront on every Space-member addition. Instead, we **insert lazily on first message receipt**:

```
On message delivery hook → for each implicit subscriber receiving the message:
  INSERT IGNORE INTO group_member
    (uid, group_no, role=common, status=1, is_implicit=1, is_external=0)
```

This amortises the row write across actual usage. A 1000-member Space with 50 public channels but where each user is active in 10 of them ends up with 10 000 rows of `group_member` (acceptable for a single-table layout up to ~100 000 rows; no partitioning required at that scale).

The `is_implicit` flag is a new column on `group_member`. It is **not** overloaded onto `status` or `role` — both of those have well-defined existing semantics (`status = blacklist`; `role = manager / common`). Mixing implicit-membership semantics into either of those columns would corrupt every existing query that filters by them.

### 3.4 Subscribers-resolution helper (the local function)

For the few callbacks and admin paths that *do* still want a "what should this channel's subscriber set look like right now?" answer (the webhook datasource fallback, the `/v1/group/members` admin API, audit tooling, integration tests), `pkg/space/group_subscribers.go: ResolveGroupSubscribers(groupNo)` returns:

```sql
-- visibility = 0
SELECT uid FROM group_member WHERE group_no = ? AND is_deleted = 0 AND status = 1

-- visibility = 1
SELECT uid FROM (
  SELECT sm.uid FROM space_member sm
    JOIN space s ON s.space_id = sm.space_id AND s.status = 1
    WHERE sm.space_id = ? AND sm.status = 1
  UNION ALL
  SELECT gm.uid FROM group_member gm
    WHERE gm.group_no = ? AND gm.is_deleted = 0 AND gm.status = 1
      AND gm.uid NOT IN (
        SELECT uid FROM space_member
          WHERE space_id = ? AND status = 1
      )
) t
```

`UNION ALL` plus an explicit `NOT IN` clause is preferred over `UNION DISTINCT` because the optimiser can hash-merge it instead of falling back to a tempfile sort, which matters at 10 000-member Space scale (the projected upper bound). At ≤ 1 000 members the difference is unmeasurable.

All identifiers (`spaceID`, `groupNo`, …) bind through `?` placeholders — there is no string interpolation. The function never accepts user-supplied `groupNo` directly without the standard 32-hex format check at the API boundary.

### 3.5 Compensation for IM-write failures

Every event hook commits its database transaction before issuing `IMAddSubscriber` / `IMRemoveSubscriber`. If the IM write fails (network blip, WuKongIM cluster instability) the database has moved forward but wkdb has not. Two mitigations:

- The IM call is wrapped in a retry-with-backoff for transient errors (existing pattern across the codebase).
- A persistent compensation queue (`group_subscriber_compensation`) records the intended `IMAddSubscriber` / `IMRemoveSubscriber` operation when the call ultimately fails. A background worker drains the queue on a 30-second cadence. This is the same pattern OCTO uses today for thread subscriber writes; we extend it rather than introduce a new mechanism.

The queue is best-effort: it does not provide cross-region durability and a planned WuKongIM outage longer than the queue's retention window (24 h) will require operational re-fanout, the same as for any other IM-side state today.

---

## 4. Data model

### 4.1 Migration: `group.visibility` column + index

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

The `ALGORITHM` / `LOCK` clauses are **mandatory**, not advisory. On MySQL 8.0 with the default `ROW_FORMAT = DYNAMIC`, an `ADD COLUMN ... SMALLINT NOT NULL DEFAULT 0` is an `INSTANT` operation and completes in milliseconds; without the explicit clause, a deployment that lands behind a long-running migration on a `COMPRESSED` table (or any table where `INSTANT` does not apply) will fall back to `INPLACE` and rebuild the table, holding metadata locks for minutes. Forcing the clause makes the migration fail loudly rather than silently degrade.

`CREATE INDEX` on a 100 000-row `group` table runs at 5–30 seconds online. Pre-flight `pt-online-schema-change` dry-run on staging is required before the production rollout.

The matching Go-side `SELECT *` queries continue to work unchanged because dbr's struct-tag binding tolerates extra columns. Existing `octo-server` instances that have not yet rolled out the new code keep functioning during the phased deploy because they ignore the new column. New code paths must be tolerant of `NULL` only on the *Down* path; on the *Up* path the `DEFAULT 0` ensures every row has a value.

### 4.2 New table: `group_visibility_change` (audit log)

```sql
CREATE TABLE IF NOT EXISTS `group_visibility_change` (
    `id` BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    `group_no` VARCHAR(40) NOT NULL,
    `from_visibility` SMALLINT NOT NULL,
    `to_visibility` SMALLINT NOT NULL,
    `operator_uid` VARCHAR(40) NOT NULL,
    `affected_uids_json` MEDIUMTEXT NULL
        COMMENT 'On 1->0 (wipe), the list of revoked uids. Null on 0->1.',
    `created_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    KEY `idx_group_no_created` (`group_no`, `created_at`)
);
```

This table is append-only by design. Operations needs the row history when answering "when did this channel become public?" or "who lost access in last week's wipe?" — both common compliance / customer-support queries.

### 4.3 New column: `group_member.is_implicit`

```sql
ALTER TABLE `group_member`
  ADD COLUMN `is_implicit` TINYINT NOT NULL DEFAULT 0
    COMMENT '1 = lazily-inserted Space-public subscription; 0 = explicit invite',
  ALGORITHM=INSTANT, LOCK=NONE;

CREATE INDEX `idx_group_member_implicit`
  ON `group_member` (`group_no`, `is_implicit`, `is_deleted`)
  ALGORITHM=INPLACE, LOCK=NONE;
```

`is_implicit = 1` means the row was lazy-inserted by the message-delivery hook because the user is an implicit subscriber (Space membership). `is_implicit = 0` (the default) is the existing behaviour and covers every row written before this migration.

The five canonical paths that **must** clean up `is_implicit = 1` rows (and the unit-test obligations on each):

| # | Trigger | Cleanup behaviour | Test coverage |
|---|---|---|---|
| 1 | `space_member.remove` (or status `1 → 0`) | Delete `is_implicit = 1` rows of this user across every `visibility = 1` channel of this Space | Unit test: leave Space, assert no surviving implicit rows for the user |
| 2 | `space.status 1 → 0` | Delete every `is_implicit = 1` row for every channel under the Space | Unit test: disable Space, assert all implicit rows for that Space are gone |
| 3 | `group.visibility 1 → 0` (wipe) | Delete every `is_implicit = 1` row for that channel | Unit test: flip back to private, assert only explicit rows remain |
| 4 | User-initiated "delete conversation" | Soft-hide via a separate session-state column; do *not* delete the `is_implicit` row (otherwise the channel reappears on the next message) | Unit test: hide and re-receive; row must persist, hide flag must filter session list |
| 5 | `group.status = dissolved` | `is_deleted = 1` cascades through `group_member` already; no special handling | Unit test: dissolve; subsequent `ExistMembers` returns empty |

The cleanup logic is centralised in a single helper `cleanupImplicitMembers(spaceID, uid, scope)` so that path #4 is the only place a different behaviour is implemented; paths #1, #2, #3, #5 share the helper. Cleanup runs in the same database transaction as the trigger event — never asynchronously — so that there is no observable window in which a revoked user can still receive messages.

### 4.4 Reuse of `channel_offset`

No new table is needed for the historical-message lower bound. The existing `channel_offset(uid, channel_id, channel_type, message_seq)` table is the single source of truth for the per-user sync starting point. The `private → public` transition writes one `channel_offset` row per newly-implicit subscriber at the current `maxSeq`. Subsequent client-side syncs (`octo-server/modules/message/api.go:1855 insertOrUpdate` is the existing write path) read the row as the lower bound, exactly as they do for `allow_view_history_msg = 0` invites today.

### 4.5 Invariants the schema must hold

- `group.space_id` is **immutable** after channel creation. Changing it would let a public channel migrate between Spaces and double-fan-out its subscribers; we forbid the operation in the API layer and document it as an invariant. There is no current code path that mutates `group.space_id` (verified by searching `modules/group/` for any `UPDATE ... SET space_id`); the invariant is therefore declarative rather than enforced by a constraint.
- `(group_member.is_implicit = 1) ⇒ (group_member.role = common ∧ group_member.is_external = 0)`. Implicit subscribers are always rank-and-file internal members. A bot, an external user, or a manager promotion always writes an explicit (`is_implicit = 0`) row.
- `(group.visibility = 1) ⇒ (group.space_id != "")`. A personal-space channel cannot become public; the visibility-flip API enforces this.

---

## 5. Security & privacy

### 5.1 Cross-Space isolation

The fanout SQL in `ResolveGroupSubscribers` (§3.4) joins `space_member` against the channel's own `space_id`. A user belonging only to Space B cannot appear in the subscriber set of a channel whose `space_id = SpaceA`, because the `WHERE sm.space_id = ?` filter rejects them. Combined with the `group.space_id`-immutability invariant (§4.5), this guarantees that public channels never bleed across Spaces.

The session-sync path (`octo-server/modules/message/api_conversation.go:295,436`) does an additional `ExistMembers` filter; combined with the per-Space `X-Space-ID` header validated by `octo-server/pkg/space/middleware.go`, a user in Space A who has cached a public channel from Space A in their client cannot have it surface when they switch their UI to Space B.

The one corner case worth documenting: a user who has both a Space-A `space_member` row *and* an explicit Space-B `group_member` row (e.g. they are an external on a Space-A channel that is also public) will continue to see the channel in their session list under Space-B. This matches the current external-group behaviour exactly; no new leakage is introduced.

### 5.2 XSS / injection on channel name, topic, notice

Channel name, topic, and notice are user-editable strings. In the existing `visibility = 0` world, a script injection in a channel name renders for the (small) explicit `group_member` set. In the `visibility = 1` world, the same payload renders for **every member of the Space**. The blast radius scales linearly with Space size.

Server-side mitigations, mandatory:

- On every `POST` / `PUT` of `name`, `topic`, `notice`, the handler must apply `html.EscapeString` *before* writing to the database. Client-side escaping is *not* sufficient — mobile and web clients must each be trusted independently, and any single one missing the escape becomes the path of least resistance for an attacker.
- Length caps are enforced at the API layer: `name ≤ 50` characters, `topic ≤ 100`, `notice ≤ 1000`.
- The same escaping is retroactively applied to existing rows during the migration window, behind a feature flag, so that the upgrade does not leave stale unescaped strings exposed in newly-public channels.

The server-side escape pattern matches what OCTO already does for system messages on the message-write path; this RFC extends it to the channel-metadata path, which historically relied on client escaping.

### 5.3 SQL injection

Every query in §3 binds all dynamic values through `?` placeholders. None of the queries interpolate `groupNo`, `spaceID`, or any user input into the SQL string. The `ResolveGroupSubscribers` helper additionally validates the `groupNo` argument against `^[0-9a-f]{32}$` at the API boundary so that a malformed input is rejected before reaching the database.

### 5.4 Blacklist behaviour

A blacklisted user must be denied access regardless of their Space membership. In `visibility = 1` channels this means the manager's blacklist action writes an explicit `group_member` row with `status = blacklist, is_implicit = 1`:

```
manager blacklists uid X on a public channel:
  INSERT INTO group_member (uid, group_no, role=common, status=blacklist, is_implicit=1)
  ON DUPLICATE KEY UPDATE status=blacklist
  IMRemoveSubscriber(groupNo, [X])
  IMRemoveSubscriber(every thread under groupNo, [X])
```

The delivery path's `Blacklist` callback (`octo-server/modules/group/1module.go:80-82`) continues to return rows with `status = blacklist`; the new `is_implicit = 1` flag is orthogonal. Subscriber-resolution must `EXCEPT` blacklisted rows from the implicit Space-member subscriber set — see §3.4 for where this filter goes.

### 5.5 Forbidden (mute-all) behaviour

When a `visibility = 1` channel has `Forbidden = 1`, only `manager` accounts may post (existing behaviour at `octo-server/modules/group/1module.go:83-94 Whitelist`, which calls `GetMemberUIDsOfManager`). Space admins are **not** auto-elevated into the post-allowlist. The deliberate consequence: a Space admin who needs to broadcast in a forbidden public channel must first be promoted to channel `manager`. We do not introduce a Space-admin-override path because doing so would conflict with the §2.2 commitment that the Space admin cannot unilaterally override channel-level decisions.

This design choice is documented in the operations runbook so that on-call engineers don't add the override "as a quick fix" during incident response.

### 5.6 Bot ownership

Bot accounts (`group_member.is_robot = 1`) interact with channels through invitations: a human user invites a bot, and the inviter's identity is recorded so that the bot's lifecycle follows the inviter (`octo-server/modules/group/db.go:702-728 QueryBotsInvitedByUIDTx`). The `visibility = 1` path **does not** treat the implicit Space-member set as bot inviters — bots are not lazily fanned out. A bot enters a public channel only via an explicit `group_member` row written by an explicit invitation flow.

This avoids a class of bugs where a Space-membership change would cascade through bot ownership ("user X left the Space, so we should remove all bots they invited" — but they didn't invite the bot, the channel did) and keeps bot lifecycle aligned with the existing private-channel semantics.

---

## 6. Rollout & operations

### 6.1 Phased delivery

The total engineering effort is approximately **21 person-days** end-to-end. The phases below are designed so that no single phase is on the critical path for every other; phases 2 and 3 (UI work) can run in parallel with phase 1 once the API contract stabilises.

| Phase | Scope | Effort |
|---|---|---|
| 1a | Migration (`group.visibility`, `group_visibility_change`, `group_member.is_implicit`); model fields; visibility-flip API; unit tests | 2 d |
| 1b | The five event hooks (visibility flip 0↔1, space-member add/remove, space-disable); thread fanout sync; compensation queue extension | 4 d |
| 1c | Lazy `is_implicit` insertion on first message receipt; the five cleanup paths; `channel_offset` lower-bound write; blacklist adaptation | 4 d |
| 1d | Integration tests covering: full Space + flip + leave + blacklist + forbidden + thread; soak test on a synthetic 1000-member Space | 2 d |
| 2 | `octo-web`: channel-settings UI for the visibility toggle; public-channel badge; session-list rendering | 4 d |
| 3 | `octo-ios` / `octo-android`: visibility badge + entry hint (no logic changes; both clients already render whatever the server says) | 2 d |
| 4 | Single-Space gated rollout; subscriber-fanout latency P99 monitoring; wkdb size-growth monitoring; gradual ramp to all production Spaces | 3 d |

There is no separate "spike" phase. The architectural questions that motivated the original v0.1 spike — *does the WuKongIM delivery path call `IDatasource.GetSubscribers`?* — are answered by reading the upstream source, not by running a probe (see the engineering-review companion).

### 6.2 Gradual rollout

Phase 4 begins with a single internal Space. Three monitoring dashboards must be green for 48 hours before the feature is ramped to the first external Space:

- **Fanout-event latency (P99)**: per-hook timing for the five event hooks. Target: P99 < 500 ms for `OnSpaceMemberAdd`, P99 < 5 s for `SwitchVisibilityToPublic` on Spaces up to 1 000 members.
- **wkdb subscriber-bucket size**: total subscriber rows in the WuKongIM cluster, broken down per channel-type. Sudden growth indicates a fanout that fired too many times (e.g. a loop in the cleanup path).
- **`is_implicit` row count**: total rows in `group_member` with `is_implicit = 1`, broken down per Space. The expected ceiling is `space_member_count × public_channel_count × active_user_fraction`; deviations indicate a leak in the cleanup paths.

After the internal Space, the rollout proceeds in three batches: a hand-picked 5 customer Spaces, then 50, then global. Each batch holds for 7 days unless an alert fires.

### 6.3 Rollback

Rollback is simply: flip every `visibility = 1` channel back to `0` via the existing API (`SwitchVisibilityToPrivate`). This wipes implicit subscribers via the standard event hook and is operationally equivalent to a customer-initiated wipe; no schema rollback is required. The migrations themselves are non-destructive and remain safe to leave in place even if the feature is later withdrawn.

The compensation queue continues to drain during rollback. Operations should monitor it for backlog growth — a long backlog at rollback time indicates that WuKongIM was unable to keep up with the flood of `IMRemoveSubscriber` calls; the right response is to throttle the rollback rather than work around the queue.

### 6.4 Telemetry

In addition to the rollout dashboards above, the following counters are emitted on every event hook (Prometheus, hourly aggregation):

```
group_visibility_flip_total{from_visibility, to_visibility}
group_space_member_event_total{event_type, public_channels_affected}
group_implicit_member_lazy_insert_total{source=delivery}
group_implicit_member_cleanup_total{trigger_path=1..5}
group_subscriber_compensation_queue_depth
```

The cleanup-path counters (`trigger_path=1..5`) are particularly load-bearing: a flatlining counter on path #1 (Space-member-leave) when the membership-change rate is non-zero is a strong indicator of a regression in the leave-handler.

---

## 7. Open questions

These are deliberately *not* blocking implementation. They are flagged here so that subsequent proposals can pick them up without re-discovering the trade-off.

1. **Ramp test at > 1000-member Space scale.** Production data on Spaces with 5 000 + active members will tell us whether the fanout latency targets in §6.2 hold. We will run a synthetic load test in phase 1d, but the real numbers come from production. If the P99 ceiling is breached, the most promising mitigation is batching multiple `IMAddSubscriber` calls per IM HTTP round-trip (the API already accepts batched subscribers, but the event-hook code calls it one Space-member at a time today).
2. **Should the client expose two conversation tabs — "joined" vs "available"?** Some workplace-IM products (Slack, Discord) split the user's channel list into channels they are explicitly part of versus channels they could join. v1 ships everything in a single list (consistent with current OCTO behaviour). Six months of usage data on `is_implicit = 1` row counts and user-initiated "delete conversation" actions will tell us whether the unified list is too noisy.
3. **Should Space admins be able to override channel visibility?** §2.2 says no. We should re-evaluate after six months of production data, in particular: how many customer-support tickets ask for "I want every channel in my Space to be public", and how many ask the inverse, "the channel admin made the wrong call and I want to override". The answer determines whether the §2.2 model is *too* restrictive.

---

## 8. References

- Originating discussion: <https://github.com/Mininglamp-OSS/community/discussions/1>
- Tracking issue: <https://github.com/Mininglamp-OSS/octo-server/issues/23>
- Engineering review: [`./0001-public-channels-review.md`](./0001-public-channels-review.md)
- Prior art: Discord public channels; Slack channel browser + `#general`; Feishu / Lark 频道; WeCom 公告群
- Related design notes:
  - `octo-server/docs/external-group-design.md` — orthogonal `is_external_group` axis
  - `octo-server/modules/thread/thread.md` — the eager-fanout pattern this RFC adapts
- WuKongIM upstream (`develop`):
  - `internal/channel/handler/event_distribute.go:337-359` — delivery-time subscriber resolution
  - `pkg/cluster/store/channel.go:57-59` — store-level subscriber read
  - `pkg/wkdb/subscriber.go:55` — pebble-backed persistence
