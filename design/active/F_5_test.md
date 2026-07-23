---
author:
id: F_5
internalId: c6b0a32d-f2a7-4b37-8652-7348162bfc8e
title: unify open document drafts and save state
status: design
owner:
affects:
agents:
  - design/activity/card__c6b0a32d-f2a7-4b37-8652-7348162bfc8e.json#conversation=agent-b7c16895-0929-4cc8-a70a-d011cb01053d
  - design/activity/card__c6b0a32d-f2a7-4b37-8652-7348162bfc8e.json#conversation=agent-d8d243e5-7305-4cb2-b7f7-060760089cfc
  - design/activity/card__c6b0a32d-f2a7-4b37-8652-7348162bfc8e.json#conversation=agent-3504f3c3-a4e7-4350-bea9-23730bf488b0
  - design/activity/card__c6b0a32d-f2a7-4b37-8652-7348162bfc8e.json#conversation=agent-f92658ef-dc8f-43b3-977c-3d4286ce7db6
policy:
after: 2f8cfca2-95da-464f-b14c-b5980a74d2d9
---

## Goal

Give each open file one canonical `OpenDocument`, shared by list and board views, that owns its draft and file-level dirty state. This unifies content, saves, renewal, and events. `dirty` means the complete JSON/Markdown file still needs successful persistence; prompt, phrase, body, and editor-instance dirty flags do not exist.

## Current problems

- `MarkdownEditorStateStore`, `ActionService`, and `CommitBatcher` duplicate file buffers, revisions, and save stages.
- Markdown data sources replace `OpenDocument` identity with string IDs, then re-resolve domain objects.
- Board/list editors can diverge on one card.
- `ActionService.savedRevision` advances when `DataService.persistActionFile()` schedules—not when `storage.commit()` succeeds.
- `CardBodySaveStatus` hides the split by combining editor and `CommitBatcher` dirty state.

## Design

### Canonical documents

`OpenFilesService` owns a stable-card/action-identity registry and list/board membership sets, all referencing canonical documents. A second view reuses a registered card without opening/activating a list tab; closing it removes only that membership.

Registration lasts while either view owns the document or unsaved/recoverable state remains. Close/project-switch flows explicitly flush, discard, or retain that state.

Each document is a stable `EventTarget` exposing its full current domain object, draft, file-level dirty state, and private edit/save revisions. Renewal updates it. Draft, dirty, saved, and renewal transitions emit events; consumers read properties directly. No boolean `getSnapshot()`.

### Shared drafts

Edits update the active document draft and dirty the whole file. Data sources retain full documents, never reconstructing ownership from paths/compound IDs. Both card editors subscribe to one document. Origin metadata updates the other binding without replacing the origin's state or losing cursor/history.

`MarkdownEditor` reports edits through its active document/data source but never marks a persisted document clean. Transient editors without an `OpenDocument` may keep `onDirtyChange`; they remain outside project-file save state.

### Action sections

An action is one `ActionOpenDocument` and one dirty file; prompt and phrases are only editor sections:

```ts
type ActionMarkdownSection =
  | { kind: 'prompt' }
  | { kind: 'phrase'; identity: string }
```

The active target pairs the canonical action document with a section. Its identity routes outgoing text, external replacement, undo history, and phrase deletion; it owns no dirty state and is not a public namespaced `markdownDocumentId`. Card body has one implicit `body` section.

Key echo suppression and binding-owned history by canonical document plus section. Closing a document discards all its histories; deleting a phrase discards only that section's.

### Save ownership

`OpenDocument.dirty` is authoritative:

1. Mutate: increment private edit revision; mark dirty.
2. Schedule: capture document, revision, and complete serialized file.
3. Batch: `CommitBatcher` tracks per-file pending/in-flight entries.
4. Commit: only successful `storage.commit()` acknowledges the captured revision.
5. Fail/stale: stay dirty; older completion cannot clear newer edits.

Unschedulable invalid action drafts stay dirty; once repaired, serialize/queue the complete current action. `ActionService` may retain draft, validation, conflict, and queue revisions, but saved revision means successful batch persistence—not `CommitBatcher` handoff.

Path changes carry the same document/revision through the batch; successful rename renews its path, not identity.

`ProjectPersistenceService` aggregates pending-save state from dirty/recoverable documents, pending/in-flight batches, and active storage operations. `hasPendingPush` stays separate Git state. Documentless changes, including board ordering/bulk operations, remain `CommitBatcher` entries.

### Removal

- Remove `MarkdownEditorStateStore`: props, instances, subscriptions, and tests.
- Make `CardBodySaveStatus` read file-level dirty state from the canonical card document, not combined sources.
- Rename generic per-file queries such as `hasPendingActionFile` if conflict handling still needs them.

## Failure and lifecycle rules

- Section/tab/view/project/branch switches and app close flush editor drafts into persistence before batches.
- Failed flushes/commits leave document and global state dirty and retryable.
- External changes renew clean drafts; dirty documents use existing conflict/recovery behavior, never silently replacing local content.
- Board/list editors share one draft; non-origin editors sync without echo writes.
- Closing the last view of an invalid/conflicted action preserves its recoverable draft.
- Reopening a fully saved, released document creates a fresh wrapper and history.

## Implementation scope

- Replace ID-only Markdown binding snapshots/events with full documents and optional section targets.
- Preserve the exact outgoing target while flushing during editor/history-monitor switches.
- Use document drafts/current paths in card/action data sources.
- Add `OpenFilesService` registry and list/board membership APIs.
- Add document/revision metadata and successful-save callbacks to `CommitBatcher` writes/renames.
- Tie action-draft acknowledgement and project persistence aggregation to batch completion.
- Move board/list components, active-card hooks, toolbar/diff consumers, cleanup handlers, and save status to full documents.
- Delete compound action Markdown ownership helpers after internal history migration.

## Acceptance criteria

- One card's board/list views share its `CardOpenDocument`, draft, dirty state, and events.
- Card body/action field/prompt/phrase edits dirty exactly one file document.
- Successful physical persistence clears only the saved revision; failures/newer edits stay dirty.
- Prompt/phrase switches write the outgoing section; the action stays one dirty document.
- Invalid/conflicted actions stay recoverable and dirty without a batch.
- Save indicators, flush guards, project/branch switches, and app close observe all dirty documents/batches.
- No `MarkdownEditorStateStore`, editor-specific persisted dirty flag, public compound `markdownDocumentId`, or pre-persistence `savedRevision` remains.
- Tests cover shared board/list edits, section routing, renewal/rename, close/reopen, invalid actions, external conflicts, failed saves, and in-flight edits.
- App lint and tests pass.

## See also

- [[J-017]]
- [[J-018]]
- [[B-052]]
- [[B-068]]
- [[B-071]]
