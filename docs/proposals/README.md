# OCTO Proposals (RFCs)

This directory hosts accepted **OCTO design proposals** — the single source of truth for cross-repo design decisions.

## What goes here

A proposal lives here once it has:

1. Started as a **Discussion** in this `community` repo (an Idea, a Q&A clarification, or an Announcement-driven design question).
2. Gone through engineering review (typically captured in a tracking issue on the relevant implementation repo, e.g. `octo-server`).
3. Reached an "Accepted" decision recorded in the proposal's `Status` field.

Implementation work then lives as issues + PRs in the **specific implementation repos** (`octo-server`, `octo-lib`, `octo-web`, `octo-admin`, etc.), each linking back to the proposal here.

## Numbering

Proposals are numbered sequentially: `NNNN-short-slug.md`, e.g. `0001-public-channels.md`.

A proposal may carry an optional companion `NNNN-<slug>-review.md` containing the engineering review report that informed the design.

## Proposal lifecycle

```
Discussion (Idea)
      │
      ▼
Draft proposal (PR to docs/proposals/)
      │
      ▼
Engineering review (tracking issue on implementation repo)
      │
      ▼
Status: Accepted   ← merged into main
      │
      ▼
Implementation issues opened on individual repos, each: "Refs <community RFC link>"
```

## Status values

- `Draft` — under discussion
- `Accepted` — design approved, implementation in progress / scheduled
- `Implemented` — shipped, doc kept for historical reference
- `Withdrawn` — abandoned or superseded (link to successor)

## Index

| # | Title | Status |
|---|-------|--------|
| [0001](./0001-public-channels.md) | Public-in-Space channels | Accepted |

## Authoring a new proposal

1. Copy [`template.md`](./template.md) to `NNNN-<slug>.md`
2. Reference the originating discussion (link)
3. Open a PR to `main`
4. Cross-link the PR from the original discussion + any tracking issues
5. Address review comments, update status to `Accepted` on merge
