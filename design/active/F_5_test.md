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

Make one canonical `OpenDocument` own each open file's in-memory draft and dirty state. List and board views share that object, preventing content, save state, renewal, and events from diverging.

Dirty means the complete JSON or Markdown file still needs successful persistence. Prompt, phrase, body, and editor-instance dirty flags do not exist.

## Current problems

- `MarkdownEditorStateStore` tracks an editor buffer while `ActionService` revisions and `CommitBatcher` separately track the same file through later save stages.
- Markdown data sources discard `OpenDocument` identity, retain string IDs, then resolve domain objects again.
- Board and list editors can hold different buffers for the same card.
- `ActionService.savedRevision` advances when `DataService.persistActionFile()` schedules a batch, before `storage.commit()` succeeds.
- `CardBodySaveStatus` masks this split by combining editor and `CommitBatcher` dirty state.

## Design

### Canonical open documents

`OpenFilesService` owns:

- a registry keyed by stable card/action identity;
- list-view and board-view membership sets that reference the same canonical `OpenDocument`.

Opening a registered card in another view reuses its document without opening or activating a list tab. Closing a view removes only its membership. The document stays registered while owned by either view or holding unsaved/recoverable state. Close and project-switch flows must explicitly flush, discard, or retain that state.

Each document is a stable `EventTarget` exposing its current full domain object, draft, file-level dirty state, and private edit/save revisions. Domain renewal updates this stable object. Draft, dirty, saved, and renewal transitions emit events; consumers read properties directly. Do not add a boolean `getSnapshot()`.

### Shared drafts

Every edit updates the active document's draft and marks the whole file dirty. Data sources retain full documents—never reconstructing ownership from paths or compound IDs. Both card editors read and subscribe to the same draft. Origin metadata lets other bindings update without making the originating editor replace its state or lose cursor/history.

`MarkdownEditor` reports edits through the active document/data source but never declares a persisted document clean. Local transient editors without an `OpenDocument` may retain `onDirtyChange`; they are outside project-file save state.

### Action sections

An action has one `ActionOpenDocument` and one dirty file. Prompt and phrases are only editor sections:

```ts
type ActionMarkdownSection =
    | { kind: 'prompt' }
    | { kind: 'phrase'; identity: string }
```

The active target pairs the canonical action document with its section. Section identity routes outgoing text, external replacement, undo history, and phrase deletion. It neither owns dirty state nor appears as a public namespaced `markdownDocumentId`. Card body has one implicit `body` section.

Echo suppression and history key on canonical document plus section; history remains binding-owned. Closing a document discards all its histories. Deleting a phrase discards only that section's history.

### Save ownership

`OpenDocument.dirty` authoritatively says whether an open file needs saving:

1. Mutation increments the private edit revision and marks the document dirty.
2. Scheduling persistence captures the document, revision, and complete serialized file.
3. `CommitBatcher` tracks operational pending/in-flight entries per file.
4. Only a successful `storage.commit()` acknowledges the captured revision.
5. Failure leaves the document dirty; completion of an older revision cannot clear newer edits.

Invalid action drafts remain dirty even when unschedulable. Once repaired, the complete current action is serialized and queued. `ActionService` may keep draft, validation, conflict, and queue revisions, but its saved revision must mean successful batch persistence—not handoff to `CommitBatcher`.

Path changes carry the same document and revision through the batch. A successful rename renews the path without replacing document identity.

`ProjectPersistenceService` remains the aggregate coordinator. It derives pending-save state from:

- dirty/recoverable documents;
- pending/in-flight commit batches;
- active storage operations.

`hasPendingPush` remains separate Git state. Files without an open document, including board ordering and bulk-operation changes, remain represented by `CommitBatcher` entries.

### Removal

Remove `MarkdownEditorStateStore`, including its props, instances, subscriptions, and tests. `CardBodySaveStatus` subscribes to the canonical card document and reads only its file-level dirty state. Rename generic per-file batch queries such as `hasPendingActionFile` where still required for conflict handling.

## Failure and lifecycle rules

- Before batches flush, editor drafts flush into persistence when switching section, tab, view, project, or branch, and when closing the app.
- A failed draft flush or storage commit leaves document and global persistence state dirty, retained, and retryable.
- External content renews a clean document's draft. For a dirty document, existing conflict/recovery behavior applies; local content is never silently replaced.
- Simultaneous board/list editing shares one draft; non-origin editors synchronize without echoing writes.
- Closing the last view of an invalid or conflicted action must retain its recoverable draft.
- Reopening a fully saved, released document creates a fresh wrapper and history.

## Implementation scope

- Replace ID-only Markdown binding snapshots/events with full-document targets and optional sections.
- Update Markdown editor and history-monitor switching so a flush retains the exact outgoing target.
- Make card/action data sources use document drafts and current document paths.
- Add canonical registry and list/board membership APIs to `OpenFilesService`.
- Add document/revision metadata and successful-save callbacks to `CommitBatcher` entries for writes and renames.
- Align action draft acknowledgement and project persistence aggregation with batch completion.
- Move board/list components, active-card hooks, toolbar/diff consumers, cleanup handlers, and save status to full documents.
- Delete compound action Markdown ownership helpers unless still needed for internal history migration.

## Acceptance criteria

- Board and list views of a card receive the same `CardOpenDocument`, draft, dirty state, and events.
- Editing a card body, action field, prompt, or phrase marks exactly one file-level document dirty.
- Successful physical batch persistence clears only its saved revision; failures and newer edits remain dirty.
- Prompt/phrase switching writes the outgoing section while the action remains one dirty document.
- Invalid/conflicted actions remain recoverable and dirty without a queued batch.
- Project save indicators, flush guards, project/branch switching, and app close observe every dirty document and pending batch.
- No `MarkdownEditorStateStore`, editor-specific persisted dirty flag, public compound `markdownDocumentId`, or premature `savedRevision` remains.
- Tests cover shared board/list editing, section routing, renewal/rename, close/reopen, invalid actions, external conflicts, failed saves, and edits during an in-flight save.
- App lint and tests pass.

## See also

- [[J-017]]
- [[J-018]]
- [[B-052]]
- [[B-068]]
- [[B-071]]
