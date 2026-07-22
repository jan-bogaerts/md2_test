---
author: 
id: F_5
internalId: c6b0a32d-f2a7-4b37-8652-7348162bfc8e
title: unify open document drafts and save state
status: new
owner: 
affects:
agents:
  - design/activity/card__c6b0a32d-f2a7-4b37-8652-7348162bfc8e.json#conversation=agent-b7c16895-0929-4cc8-a70a-d011cb01053d
policy:
after: a5e5e021-323f-4b63-b2c5-e7f9bc54a188
---
## Goal

Use one canonical `OpenDocument` per open file for its in-memory draft and dirty state. List and board views share it, so content, save state, renewal, and events cannot diverge. Dirty is file-level: the complete JSON or Markdown file still needs successful persistence; prompt, phrase, body, and editor-instance dirty flags do not exist.

## Problems

`MarkdownEditorStateStore`, `ActionService` revisions, and `CommitBatcher` independently track the same file across save stages. Markdown data sources discard `OpenDocument` identity for string IDs and later resolve domain objects again, allowing board and list editors to keep different buffers. `ActionService.savedRevision` advances when `DataService.persistActionFile()` schedules a batch rather than when `storage.commit()` succeeds. `CardBodySaveStatus` masks the split by combining editor and batcher dirty state.

## Model and invariants

### Canonical documents and view ownership

`OpenFilesService` owns a registry keyed by stable card/action identity and separate list/board membership sets, both referencing the same canonical `OpenDocument`.

- Opening a registered card in another view reuses its document without opening or activating a list tab.
- Closing a view removes only that membership.
- A document remains registered while either view owns it or it has unsaved/recoverable state. Closing or switching projects must explicitly flush, discard, or retain that state.
- Reopening a fully saved, released document creates a fresh wrapper and history.

Each document is a stable `EventTarget` exposing its full current domain object, draft, file-level dirty state, and private edit/save revisions. Domain renewal updates that object. Draft, dirty, saved, and renewal transitions emit events; consumers read properties directly—do not add `getSnapshot()` for a boolean.

### Drafts and editor bindings

Every edit updates the active document draft and dirties the whole file. Data sources retain full documents instead of reconstructing ownership from paths or compound IDs. Board and list editors subscribe to the same draft. Origin metadata keeps the source editor's state, cursor, and history intact; other editors synchronize without echoing changes back.

`MarkdownEditor` reports edits through the active document/data source but never marks a persisted document clean. Local transient editors without an `OpenDocument` may retain `onDirtyChange`; they remain outside project-file save state.

An action has one `ActionOpenDocument` and one dirty file. Prompt and phrases are only editor sections:

```ts
type ActionMarkdownSection =
    | { kind: 'prompt' }
    | { kind: 'phrase'; identity: string }
```

The active target pairs the canonical action document with a section. Section identity routes outgoing text, external replacement, undo history, and phrase deletion; it owns no dirty state and is not a public namespaced `markdownDocumentId`. Card body has one implicit `body` section.

History and echo suppression key on canonical document plus section; history remains binding-owned. Closing a document discards all its histories, while deleting a phrase discards only that section's. Section switches flush through the exact outgoing target.

### Save ownership

`OpenDocument.dirty` is authoritative for whether an open file needs saving:

1. Mutation increments a private edit revision and marks the document dirty.
2. Scheduling persistence captures the document, revision, and complete serialized file.
3. `CommitBatcher` tracks operational pending/in-flight entries per file.
4. Only a successful `storage.commit()` acknowledges that revision on the document.
5. Failure leaves it dirty; completion of an older revision cannot clear newer edits.

Invalid action drafts stay dirty even when unschedulable. Once repaired, the complete current action is serialized and queued. `ActionService` may retain draft, validation, conflict, and queue revisions, but its saved revision means successful batch persistence—not handoff to `CommitBatcher`.

Path changes carry the same document and revision through the batch. A successful rename renews the path without replacing document identity.

`ProjectPersistenceService` remains the aggregate coordinator. Pending-save state combines dirty/recoverable documents, pending/in-flight batches, and active storage operations. `hasPendingPush` remains separate Git state. Changes without an open document, including board ordering and bulk operations, remain `CommitBatcher` entries.

## Lifecycle and failure behavior

- Switching section, tab, view, project, or branch—or closing the app—flushes editor drafts into persistence before flushing batches.
- Failed flushes or commits leave document and global persistence state dirty, retained, and retryable.
- External changes renew a clean document's draft. Against a dirty document, they use existing conflict/recovery behavior and never silently replace local content.
- Closing the last view of an invalid or conflicted action retains its recoverable draft.

## Delivery checklist

- Add the canonical registry and list/board membership APIs to `OpenFilesService`.
- Replace ID-only Markdown binding snapshots/events with full-document targets plus optional sections.
- Update Markdown editor/history switching to retain the exact outgoing target during flush.
- Make card/action data sources use document drafts and current document paths.
- Add document/revision metadata and success acknowledgements to `CommitBatcher` writes and renames.
- Align action saved revisions and project persistence aggregation with batch completion.
- Migrate board/list components, active-card hooks, toolbar/diff consumers, cleanup handlers, and save status to full documents.
- Remove `MarkdownEditorStateStore` and its props, instances, subscriptions, and tests.
- Make `CardBodySaveStatus` read the canonical card document's file-level dirty state instead of combining two dirty sources.
- Rename generic per-file batch queries such as `hasPendingActionFile` where conflict handling still needs them.
- Remove compound action Markdown ownership helpers once no internal history migration needs them.
- Verify save indicators, flush guards, project/branch switching, and app close observe every dirty document and pending batch.
- Test shared board/list editing, section routing, renewal/rename, close/reopen, invalid actions, external conflicts, failed saves, and edits during an in-flight save.
- Confirm no editor-specific persisted dirty flag, public compound `markdownDocumentId`, or pre-persistence `savedRevision` remains.
- Pass app lint and tests.

## See also

- [[J-017]]
- [[J-018]]
- [[B-052]]
- [[B-068]]
- [[B-071]]
