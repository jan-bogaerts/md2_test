---
author:
id: F_5
internalId: c6b0a32d-f2a7-4b37-8652-7348162bfc8e
title: unify open document drafts and save state
status: design
owner:
affects:
agents:
  - design/activity/card__c6b0a32d-f2a7-4b37-8652-7348162bfc8e.json#conversation=agent-671bfcc5-278b-4290-a775-69536e6887cb
  - design/activity/card__c6b0a32d-f2a7-4b37-8652-7348162bfc8e.json#conversation=agent-8630cac5-c688-46b7-8820-05f4d54a4cd0
policy:
after: 2f8cfca2-95da-464f-b14c-b5980a74d2d9
---

## Goal

Use one canonical `OpenDocument` for each open file's draft and dirty state. List and board views share it, preventing content, save state, renewal, and event divergence.

“Dirty” means the complete JSON or Markdown file still needs successful persistence. Prompt, phrase, body, and editor-instance dirty flags do not exist.

## Problems

- `MarkdownEditorStateStore`, `ActionService`, and `CommitBatcher` separately track the same file across save stages.
- Markdown data sources discard `OpenDocument` identity for string IDs, then resolve domain objects again.
- Board and list editors can hold different buffers for one card.
- `ActionService.savedRevision` advances when `DataService.persistActionFile()` schedules a batch, before `storage.commit()` succeeds.
- `CardBodySaveStatus` masks this split by combining editor and batch dirty state.

## Design

### Canonical documents and view ownership

`OpenFilesService` owns a stable card/action identity registry and list/board membership sets referencing canonical `OpenDocument`s.

- Opening a registered card in another view reuses its document without opening or activating a list tab.
- Closing a view removes only its membership.
- A document stays registered while a view owns it or it holds unsaved/recoverable state. Close and project-switch flows explicitly flush, discard, or retain that state.
- Reopening a fully saved, released document creates a new wrapper and history.

Each document is a stable `EventTarget` exposing its domain object, draft, file-level dirty state, and private edit/save revisions. Renewal updates the object without replacing the document. Draft, dirty, saved, and renewal transitions emit events; consumers read properties directly—do not add a boolean `getSnapshot()`.

### Shared drafts and editor bindings

Every edit updates the active document draft and dirties the file. Data sources retain full documents instead of reconstructing ownership from paths or compound IDs. Board and list editors share its draft and subscription. Origin metadata preserves the source editor's state, cursor, and history while preventing update echoes.

`MarkdownEditor` reports edits through its active document/data source but never declares it clean. Transient editors without an `OpenDocument` may retain `onDirtyChange`; they are outside project-file persistence.

An action has one `ActionOpenDocument` and one dirty file. Prompt and phrases are editor sections:

```ts
type ActionMarkdownSection =
  | { kind: 'prompt' }
  | { kind: 'phrase'; identity: string }
```

The active target pairs the action document with a section. Section identity routes outgoing text, external replacement, undo history, and phrase deletion, but owns no dirty state and is not a public namespaced `markdownDocumentId`. Card body has one implicit `body` section.

Echo suppression and binding-owned history key by document plus section. Closing a document discards all its histories; deleting a phrase discards only that section's. Switching retains the exact outgoing target during flush.

### Persistence

`OpenDocument.dirty` is authoritative for whether an open file needs saving:

1. Mutation increments the private edit revision and marks the document dirty.
2. Scheduling persistence captures the document, revision, and complete serialized file.
3. `CommitBatcher` tracks operational pending/in-flight entries per file.
4. Only a successful `storage.commit()` acknowledges that revision on the document.
5. Failure leaves it dirty; completion of an older revision cannot clear newer edits.

Unschedulable invalid action drafts remain dirty. Once repaired, serialize and queue the complete action. `ActionService` may keep draft, validation, conflict, and queue revisions, but its saved revision means successful batch persistence—not batch handoff.

Path changes carry the document and revision through the batch. A successful rename renews the path without replacing document identity.

`ProjectPersistenceService` remains the aggregate coordinator, deriving pending-save state from:

- dirty/recoverable documents;
- pending or in-flight batches; and
- active storage operations.

`hasPendingPush` stays separate Git state. `CommitBatcher` still represents files without open documents, including board ordering and bulk operations.

### Lifecycle and conflicts

- Switching section, tab, view, project, or branch—and closing the app—flushes editor drafts before batches.
- Failed flushes or commits leave document and global state dirty, retained, and retryable.
- External changes renew a clean draft; dirty documents use existing conflict/recovery behavior and never silently lose local content.
- Closing the last view of an invalid or conflicted action must preserve its recoverable draft.

## Implementation

- Add `OpenFilesService` canonical-registry and list/board-membership APIs.
- Replace ID-only Markdown binding snapshots/events with document targets and optional sections.
- Update editor/history target switching to flush the exact outgoing target.
- Make card/action data sources use document drafts and current document paths.
- Add document/revision metadata and successful-save callbacks to `CommitBatcher` writes/renames.
- Align action saved-revision acknowledgement and persistence aggregation with batch completion.
- Migrate board/list components, active-card hooks, toolbar/diff consumers, cleanup handlers, and save status to full documents.
- Make `CardBodySaveStatus` read only the canonical card document's file-level dirty state.
- Remove `MarkdownEditorStateStore`, including props, instances, subscriptions, and tests.
- Rename generic per-file batch queries such as `hasPendingActionFile` where still required for conflict handling.
- Delete compound action Markdown ownership helpers once no internal history migration needs them.

## Acceptance criteria

- Board and list share one `CardOpenDocument`, draft, dirty state, and event stream.
- Editing any card body, action field, prompt, or phrase dirties exactly one file document.
- Only successful physical persistence clears its captured revision; failures and newer edits stay dirty.
- Prompt/phrase switching writes the outgoing section while keeping one dirty action document.
- Invalid/conflicted actions remain recoverable and dirty without a queued batch.
- Save indicators, flush guards, project/branch switches, and app close include all dirty documents and pending batches.
- No `MarkdownEditorStateStore`, persisted editor dirty flag, public compound `markdownDocumentId`, or prematurely advanced `savedRevision` remains.
- Tests cover shared board/list editing, section routing, renewal/rename, close/reopen, invalid actions, external conflicts, failed saves, and edits during an in-flight save.
- App lint and tests pass.

## See also

- [[J-017]]
- [[J-018]]
- [[B-052]]
- [[B-068]]
- [[B-071]]
