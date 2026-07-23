---
author:
id: F_5
internalId: c6b0a32d-f2a7-4b37-8652-7348162bfc8e
title: unify open document drafts and save state
status: design
owner:
affects:
agents:
policy:
after: 2f8cfca2-95da-464f-b14c-b5980a74d2d9
---

## Goal

Use one canonical `OpenDocument` per open file across list and board views, sharing its draft, file-level dirty state, renewal, and events.

“Dirty” means its complete JSON/Markdown needs successful persistence. Prompts, phrases, bodies, and editors have no dirty flags.

## Problems

- `MarkdownEditorStateStore`, `ActionService`, and `CommitBatcher` duplicate state.
- Markdown data sources discard `OpenDocument` identity for string IDs, then re-resolve it.
- Board and list editors can hold different buffers for one card.
- `ActionService.savedRevision` advances when `DataService.persistActionFile()` schedules, before `storage.commit()` succeeds.
- `CardBodySaveStatus` hides this by combining editor/batch dirty state.

## Design

### Canonical documents

`OpenFilesService` owns the canonical card/action registry and list/board memberships:

- Opening a registered card elsewhere reuses its document without opening/activating a list tab.
- Closing a view removes only its membership.
- Registration lasts while a view owns the document or it has unsaved/recoverable state. Close/project-switch flows explicitly flush, discard, or retain it.
- Reopening a fully saved, released document creates a new wrapper/history.

Each document is a stable `EventTarget` exposing its domain object, draft, and file-level dirty state; edit/save revisions are private. Renewal updates the object while preserving the document. Draft, dirty, saved, and renewal transitions emit events. Consumers read properties directly—no boolean `getSnapshot()`.

### Drafts and editor bindings

Edits update the active draft and dirty its file. Data sources retain full documents instead of reconstructing ownership from paths/compound IDs. Board/list share the draft/subscription. Origin metadata prevents echoes while preserving the source editor's state, cursor, and history.

`MarkdownEditor` reports through its active document/data source but never declares it clean. Transient editors lacking an `OpenDocument` may keep `onDirtyChange` outside project-file persistence.

An action has one `ActionOpenDocument` and one dirty file. Its Markdown sections are:

```ts
type ActionMarkdownSection =
  | { kind: 'prompt' }
  | { kind: 'phrase'; identity: string }
```

An active target pairs document and section. The section routes outgoing text, external replacement, undo history, and phrase deletion, but owns no dirty state and is not a public namespaced `markdownDocumentId`. Cards have one implicit `body` section.

Echo suppression and binding history key on document plus section. Document close discards all its histories; phrase deletion discards only its section's. Switching preserves the exact outgoing flush target.

### Persistence

`OpenDocument.dirty` alone determines whether its file needs saving:

1. Mutation increments the private edit revision and marks it dirty.
2. Scheduling captures document, revision, and the complete serialized file.
3. `CommitBatcher` tracks pending/in-flight entries per file.
4. Only successful `storage.commit()` acknowledges that revision.
5. Failure stays dirty; older completions cannot clear newer edits.

Unschedulable invalid action drafts stay dirty; once repaired, queue the complete serialized action. `ActionService` may keep draft, validation, conflict, and queue revisions, but its saved revision means successful batch persistence, not handoff.

Path changes carry document/revision through the batch; successful rename renews the path without changing identity.

`ProjectPersistenceService` coordinates aggregate pending-save state from:

- dirty/recoverable documents,
- pending/in-flight batches,
- active storage operations.

`hasPendingPush` is separate Git state. `CommitBatcher` also covers files without open documents: board ordering and bulk operations.

### Lifecycle and conflicts

- Switching section/tab/view/project/branch or closing the app flushes editor drafts before batches.
- Failed flushes/commits leave document/global state dirty, retained, and retryable.
- External changes renew clean drafts; dirty documents use existing conflict/recovery without silently losing local content.
- Closing an invalid or conflicted action's last view preserves its recoverable draft.

## Implementation

- Add registry and list/board membership APIs to `OpenFilesService`.
- Replace ID-only Markdown binding snapshots/events with document/section targets.
- Flush the exact outgoing editor/history target on switches.
- Use document drafts/current paths in card/action data sources.
- Add document/revision metadata and success callbacks to `CommitBatcher` writes/renames.
- Acknowledge action saved revisions and aggregate persistence after batch completion.
- Migrate board/list components, active-card hooks, toolbar/diff consumers, cleanup handlers, and save status to documents. `CardBodySaveStatus` reads only the canonical card's file-level dirty state.
- Remove `MarkdownEditorStateStore`: props, instances, subscriptions, and tests.
- Rename remaining generic per-file batch queries such as conflict-related `hasPendingActionFile`.
- Delete compound action Markdown ownership helpers once internal history migration no longer needs them.

## Acceptance criteria

- Board/list share one `CardOpenDocument`, draft, dirty state, and event stream.
- Any card body, action field, prompt, or phrase edit dirties exactly one file document.
- Only successful physical persistence clears its revision; failures and newer edits remain dirty.
- Prompt/phrase switches write the outgoing section while keeping one dirty action document.
- Invalid/conflicted actions remain dirty and recoverable without a queued batch.
- Save indicators, flush guards, project/branch switches, and app close cover every dirty document and pending batch.
- No `MarkdownEditorStateStore`, persisted editor dirty flag, public compound `markdownDocumentId`, or premature `savedRevision` remains.
- Tests cover shared board/list editing, section routing, renewal/rename, close/reopen, invalid actions, external conflicts, failed saves, and in-flight-save edits.
- App lint and tests pass.

## See also

- [[J-017]]
- [[J-018]]
- [[B-052]]
- [[B-068]]
- [[B-071]]
