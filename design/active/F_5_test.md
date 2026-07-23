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
policy:
after: 2f8cfca2-95da-464f-b14c-b5980a74d2d9
---
## Goal

Make one canonical `OpenDocument` own each open file's in-memory draft and dirty state. Board and list views share that object, so content, save state, renewal, and events cannot diverge.

“Dirty” means the complete JSON or Markdown file still needs successful persistence. Prompts, phrases, bodies, and editor instances have no independent persisted-dirty state.

## Problems addressed

- Three layers track the same file inconsistently: `MarkdownEditorStateStore` owns an editor buffer, `ActionService` owns revisions, and `CommitBatcher` owns later save stages.
- Markdown data sources discard `OpenDocument` identity, keep string IDs, and re-resolve domain objects.
- Board and list editors can hold different buffers for one card.
- `ActionService.savedRevision` advances when `DataService.persistActionFile()` schedules a batch rather than after `storage.commit()` succeeds.
- `CardBodySaveStatus` masks these splits by combining editor and batcher dirty state.

## Design

### Canonical documents and shared drafts

`OpenFilesService` owns a registry keyed by stable card/action identity, with list-view and board-view membership sets that both reference the canonical `OpenDocument`.

- Opening a registered card in another view reuses its document without opening or activating a list tab.
- Closing a view removes only its membership.
- A document remains registered while owned by either view or holding unsaved/recoverable state. Close and project-switch flows must explicitly flush, discard, or retain that state.
- Reopening a fully saved, released document creates a fresh wrapper and history.

Each document is a stable `EventTarget` exposing its current full domain object, draft, file-level dirty state, and private edit/save revisions. Renewal updates the stable object. Draft, dirty, saved, and renewal transitions emit events; consumers read properties directly rather than using a boolean `getSnapshot()`.

Every edit updates the active document draft and marks the file dirty. Data sources retain full documents instead of reconstructing ownership from paths or compound IDs. Board and list editors read and subscribe to the same draft. Origin metadata lets other bindings synchronize without echoing the edit or replacing the origin's state, cursor, or history.

`MarkdownEditor` reports edits through its active document/data source but never declares a persisted document clean. Transient editors without an `OpenDocument` may retain `onDirtyChange`; they remain outside project-file save state.

### Action sections

An action has one `ActionOpenDocument` and one dirty file. Prompt and phrases are only editor sections:

```ts
type ActionMarkdownSection =
    | { kind: 'prompt' }
    | { kind: 'phrase'; identity: string }
```

The active target pairs the canonical action document with a section. Section identity routes outgoing text, external replacement, undo history, and phrase deletion, but never owns dirty state or appears as a public namespaced `markdownDocumentId`. Card bodies use one implicit `body` section.

History and echo suppression are keyed by canonical document plus section; history remains binding-owned. Closing a document discards all its histories, while deleting a phrase discards only that section.

### Save ownership

`OpenDocument.dirty` authoritatively indicates whether an open file needs saving:

1. Mutation increments the private edit revision and marks the document dirty.
2. Scheduling persistence captures the document and revision with the complete serialized file.
3. `CommitBatcher` tracks operational pending/in-flight entries per file.
4. Only successful `storage.commit()` acknowledges the captured revision.
5. Failure leaves the document dirty, and completion of an older revision cannot clear newer edits.

Invalid action drafts remain dirty even when unschedulable. Once repaired, the complete current action is serialized and queued. `ActionService` may retain draft, validation, conflict, and queue revisions, but its saved revision must mean successful batch persistence, not handoff to `CommitBatcher`.

Writes and path changes carry the same document and revision through the batch. A successful rename renews the path without replacing document identity.

`ProjectPersistenceService` remains the aggregate coordinator. Pending-save state includes dirty/recoverable documents, pending/in-flight batches, and active storage operations. `hasPendingPush` remains separate Git state. Changes without an open document—such as board ordering or bulk operations—remain represented by `CommitBatcher` entries.

### Lifecycle and failure rules

- Switching section, tab, view, project, or branch, and closing the app, flushes editor drafts into persistence before batches.
- Failed flushes or commits leave document and global persistence state dirty and retryable.
- External changes renew a clean document's draft. For a dirty document, existing conflict/recovery behavior applies and local content is never silently replaced.
- Closing the last view of an invalid or conflicted action preserves its recoverable draft.

## Implementation scope

- Replace ID-only Markdown binding snapshots/events with full documents and optional section targets.
- Preserve the exact outgoing target when Markdown editor or history-monitor switching flushes.
- Make card/action data sources use document drafts and current document paths.
- Add the canonical registry and list/board membership APIs to `OpenFilesService`.
- Add document/revision metadata and post-success callbacks to `CommitBatcher` entries for writes and renames.
- Align action save acknowledgement and project persistence aggregation with batch completion.
- Move board/list components, active-card hooks, toolbar/diff consumers, cleanup handlers, and save status to full documents.
- Remove `MarkdownEditorStateStore` and its props, instances, subscriptions, and tests.
- Make `CardBodySaveStatus` read only the canonical card document's file-level dirty state.
- Rename remaining generic per-file batch queries such as `hasPendingActionFile` where conflict handling still needs them.
- Delete compound action Markdown ownership helpers once no internal history migration needs them.

## Acceptance criteria

- Board and list views receive the same `CardOpenDocument`, draft, dirty state, and events.
- Editing any card body, action field, prompt, or phrase dirties exactly one file-level document.
- Only successful physical batch persistence clears the saved revision; failures and newer edits remain dirty.
- Prompt/phrase switching always writes the outgoing section while the action remains one dirty document.
- Invalid/conflicted actions remain dirty and recoverable without a queued batch.
- Save indicators, flush guards, project/branch switching, and app close observe every dirty document and pending batch.
- No `MarkdownEditorStateStore`, editor-specific persisted-dirty flag, public compound `markdownDocumentId`, or pre-persistence `savedRevision` remains.
- Tests cover shared board/list editing, section routing, renewal/rename, close/reopen, invalid actions, external conflicts, failed saves, and edits during an in-flight save.
- App lint and tests pass.

## See also

- [[J-017]]
- [[J-018]]
- [[B-052]]
- [[B-068]]
- [[B-071]]
