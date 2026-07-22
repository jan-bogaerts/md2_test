---
author:
id: J_1
internalId: f1150d6f-4ad8-454f-afa2-9e386d645b46
title: card commit diff viewer
status: new
owner:
affects:
agents:
  - design/activity/card__f1150d6f-4ad8-454f-afa2-9e386d645b46.json#conversation=agent-f1f52952-5d50-4c70-82e4-226c4092adb1
  - design/activity/card__f1150d6f-4ad8-454f-afa2-9e386d645b46.json#conversation=agent-7a77e441-a19f-403a-85cc-996d487fbe6e
policy:
after: ec632bef-01bd-4e16-ae1a-402cbc2d26c4
---

## Goal

From a card, list every Git commit produced by a root action run started on that card. Selecting one replaces the editor with the card-body diff and provides access to other files changed by the commit.

## Source note

`design/architecture/initial description/card_dif.md` specifies a card-popup diff icon when commits exist; its menu lists all run commits as `action + short date/time`. The Markdown editor should support diffs; `react-diff-viewer-continued` is also available.

## Current state

- F_055/F_056 collect `CommitReference[]` for one root chain, including linked `onBefore`/`on`/`onAfter` actions.
- History is stored in ignored, path-derived, per-action `logs` files. This is per-worktree and has unstable identity that can collide after moves/normalization.
- Conversation files are created before execution and rewritten about every 250 ms, although the live UI renders in-memory events.
- `ActionContext` contains the card path, not stable `header.internalId`.
- `generateDiff(commitReference)` uses `project.diffCommand`; `DiffView` renders whole-commit diffs.
- Neither the card popup nor file-mode toolbar has a card commit control.
- The Markdown editor does not register MDXEditor's `diffSourcePlugin`.

No existing data or compatibility migration must be supported.

## Project activity storage

Activity is committed project data visible to all collaborators. Replace ignored `logs` persistence with tracked files in the **primary checkout**:

| Origin | File |
| --- | --- |
| Card | `<projectFolder>/activity/card__<cardInternalId>.json` |
| Project | `<projectFolder>/activity/project.json` |

A card file owns all action-run and conversation activity for that card. Ownership uses only `header.internalId`, never path, title, display id, or a normalized path prefix.

The renderer adds `cardInternalId` to every card/file `ActionContext`; a card-origin run without it fails before starting. The root run snapshots its origin once, and children inherit it even in another worktree.

Each completed root run appends one `ActionActivityRecord` containing at least:

- `executionId`, start/completion timestamps, and terminal status;
- root action `id` and snapshotted `label`;
- origin kind and, for card origin, `cardInternalId`;
- root and child commit references in execution order;
- conversation ids created by the run.

Each commit reference stores the full hash, timestamp, source branch, changed paths/counts, and the performer action id/label when it differs from the root. Card ownership never uses a filesystem path. Machine-local absolute `repositoryRoot` paths must not be required in tracked data.

All activity I/O uses the primary project root, including runs in linked worktrees. A project-scoped writer serializes read-modify-write, writes atomically, then calls `commitTrackedPaths` for only the changed activity file. This happens after action-commit collection, and the activity commit is not added to its own record.

## Conversation persistence

Keep running conversations in `AgentRunnerService` memory and stream structured events directly to the live UI. Persist a conversation once, to its activity file, when the turn becomes `completed`, `failed`, or `cancelled`; finish the atomic write before publishing terminal `closed`. Remove initial and throttled writes.

A hard crash may lose an unfinished turn. Terminal turns remain persisted; only they can be continued. Concurrent turns for one conversation remain forbidden.

## Commit visibility

Linked worktree commits are accessible from the primary checkout through Git's shared object database, so run diff commands there using the stored hash.

A stored commit is visible when it is:

1. reachable from the primary branch `HEAD`; or
2. before merge, reachable from the source branch of a currently valid linked worktree.

If that worktree becomes invalid/unregistered while the commit is unreachable from primary `HEAD`, hide it as discarded but retain its record. This supports only the app's normal full-branch merge flow—not patch, rebase, squash, or cherry-pick inference. Ancestry only distinguishes merged from discarded worktree commits after the worktree disappears.

Re-evaluate visibility on activity load and after project/worktree state changes. If an otherwise visible Git object is unavailable locally, retain its row and report the problem when selected.

## Commit menu

- Use a `SourceCommit` icon in the card-popup header between Dirty/Saved and Close, and beside Agents/Properties in file-mode `ListEditorToolbarControls`.
- Hide it with no visible commits; show a count badge above one.
- Sort newest first across activity records. Primary label: `<root action label> · <short date & time>` in the user's locale. Secondary label: short hash and `+insertions/−deletions`; row `title`: full hash.
- Reload after a matching `cardInternalId` action finishes and after worktree/project state changes.
- Retain every record. A reasonable UI limit (for example, 50) may hide explicitly disclosed older entries; paging is unnecessary.

## Picking a commit

Selecting a commit enters diff mode on that card surface:

- Compare the card at `<commit>^` and `<commit>`, parsing each through `markdownParsingService.parse(content).body` to exclude frontmatter.
- Visually replace the live editor with a separate read-only `MarkdownEditor` using `diffSourcePlugin` and `viewMode: 'diff'`. Historical content must never enter the live editor or its dirty, autosave, or history services.
- Above it, show the root action label, full timestamp, short hash, and **Exit diff**.
- First `Escape` exits diff mode but keeps the popup open; second `Escape` closes it. Closing the popup also clears diff mode.

If the commit did not touch the card's current path, skip body diff mode and open the other-files list (rename tracking is out of scope).

Below the body diff, **Also changed (n)** lists other changed paths. Selecting one opens the existing whole-commit `DiffView`, scrolled to that file, retaining its `openInEditor` line-click behavior.

## Bridge and renderer

Add these action-bridge methods through preload, local dispatch, `ElectronActionBridge`, and the remote-control proxy:

```ts
loadCardActivity({ cardInternalId }): Promise<CardActivityFile>
readFileAtCommit({ commit, path, parent }): Promise<{ content: string; exists: boolean }>
```

`loadCardActivity` reads and validates the card's primary-checkout activity file, filters commit visibility, and returns activity for that stable identity. Malformed activity produces a visible load error, not empty history.

`readFileAtCommit` runs `git show <commit>[^]:<path>` at the primary repository root. A missing path—or missing parent for a root commit—returns `{ content: '', exists: false }`. Git paths are repository-relative and use existing root/path validation.

Renderer responsibilities:

- `card_commit_history.ts`: load/validate activity; expose no commits without an execution bridge.
- `use_card_commits.ts`: key by `cardInternalId`; refresh on matching root completion and project/worktree changes; discard stale async results after card change/unmount.
- Share `CardCommitMenu` and `CardCommitDiffPanel` between popup and file mode, with independent selection state per surface.
- Use a separate read-only historical editor. Keeping the live editor mounted is optional, but historical content must never enter its state or persistence pipelines.

## Edge cases

| Case | Required behavior |
| --- | --- |
| Root run has no commits | Record the run/conversation; hide the icon. |
| Linked actions commit on different branches | Keep one root record; evaluate each commit separately. |
| Worktree commit before merge | Show while its worktree is valid/registered. |
| Worktree removed after full merge | Keep visible because it is reachable from primary `HEAD`. |
| Worktree removed without merge | Hide, but retain the activity record. |
| Commit object unavailable locally | Keep an otherwise visible row; selecting it says `Commit is no longer available in this repository`. |
| Root commit | Treat old body as empty and render the new body as added. |
| Frontmatter-only commit | Show `No body changes in this commit`, then other changed files. |
| Card renamed after commit | If the current path exists at neither revision, show only other files. |
| Card deleted/surface closed while loading | Discard the result. |
| Two local runs update one card | Serialize writes and retain separate `executionId` records. |
| Collaborators change the same tracked file on different branches | Normal Git conflict handling applies. |

## Acceptance criteria

All requirements above are acceptance criteria. Automated tests must cover:

- stable-id storage, root/child aggregation, and serialized atomic writes;
- one terminal conversation write;
- primary/worktree visibility, discarded-worktree hiding, and full-merge retention;
- missing commits, root commits, and frontmatter-only diffs;
- Escape order and both toolbar hosts;
- the remote proxy, including other-file diffs and VS Code line opening;
- no live-editor persistence side effects.

## Documentation impact

Update F_050/F_012 conversation-persistence wording and F_056 history ownership/storage wording. No migration or legacy-schema support is required.

## See also

- `design/architecture/initial description/card_dif.md`
- `design/feature_descriptions/F_055_agent_file_change_tracking.md`
- `design/feature_descriptions/F_056_root_action_commit_history.md`
- `design/feature_descriptions/ready/F_050_one_shot_agent_conversations.md`
- `app/src/components/actions/action_commit_dropdown.tsx`
- `app/src/components/actions/diff_view.tsx`
- `desktop/src/git/diff_service.js`
- `desktop/src/git/worktree_service.js`
