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

**## Goal**

Make one canonical \`OpenDocument\` own each open file's in-memory draft and file-level dirty state. List and board views share it, preventing divergence in content, saving, renewal, and events. Dirty means the complete JSON/Markdown file still needs successful persistence; prompt, phrase, body, and editor-instance dirty flags do not exist.

**## Current problems**

\- \`MarkdownEditorStateStore\`, \`ActionService\`, and \`CommitBatcher\` track the same file through separate buffers/revisions and save stages.
\- Markdown data sources discard \`OpenDocument\` identity, retain string IDs, and resolve domain objects again.
\- Board and list card editors can hold different local buffers.
\- \`ActionService.savedRevision\` advances when \`DataService.persistActionFile()\` schedules a batch, before \`storage.commit()\` succeeds.
\- \`CardBodySaveStatus\` masks the split by combining editor and \`CommitBatcher\` dirty state.

**## Design**

**### Canonical open documents**

\`OpenFilesService\` owns a registry keyed by stable card/action identity and two membership sets: list-view and board-view documents. Both reference the same canonical \`OpenDocument\`. Opening a registered card in another view reuses it without opening or activating a list tab; closing one view removes only that membership. Keep a document registered while either view owns it or it has unsaved/recoverable state. Close and project-switch flows must explicitly flush, discard, or retain that state.

Each document is a stable \`EventTarget\` exposing its full current domain object, draft, file-level dirty state, and private edit/save revisions. Domain renewal updates the stable object. Draft, dirty, saved, and renewal transitions emit document events; consumers read properties directly. Do not add a boolean \`getSnapshot()\`.

**### Shared drafts**

Every edit updates the active document's draft and marks the whole file dirty. Data sources receive and retain full documents, never reconstruct ownership from paths or compound IDs. Both card editors read and subscribe to the same document; origin metadata synchronizes the other binding without making the originating editor replace its own state or lose cursor/history.

\`MarkdownEditor\` reports edits through the active document/data source but never declares a persisted document clean. Transient editors without an \`OpenDocument\` may retain \`onDirtyChange\`; they are outside project-file save state.

**### Action sections**

An action is one \`ActionOpenDocument\` and one dirty file. Prompt and phrases are editor sections only:

```ts
type ActionMarkdownSection =
  | { kind: 'prompt' }
  | { kind: 'phrase'; identity: string }
```

The active target pairs the canonical action document with its section. Section identity routes outgoing text, external replacement, undo history, and phrase deletion; it never owns dirty state and is not a public namespaced \`markdownDocumentId\`. Card body uses one implicit \`body\` section.

Key history and echo suppression by canonical document plus section; history remains binding-owned. Closing a document discards its histories; deleting a phrase discards only that section.

**### Save ownership**

\`OpenDocument.dirty\` is authoritative:

1. A mutation increments the private edit revision and marks the document dirty.
2. Persistence scheduling captures the document, revision, and complete serialized file.
3. \`CommitBatcher\` tracks operational pending/in-flight entries per file.
4. Only successful \`storage.commit()\` acknowledges the captured revision.
5. Failed commits leave the document dirty; an older completion cannot clear newer edits.

Invalid action drafts remain dirty even when unschedulable. Once repaired, serialize and queue the complete current action. \`ActionService\` may retain draft, validation, conflict, and queue revisions, but saved revision means successful batch persistence—not handoff to \`CommitBatcher\`.

Path changes carry the same document and revision through the batch; a successful rename renews its path without replacing its identity.

\`ProjectPersistenceService\` remains aggregate coordinator. It derives pending-save state from dirty/recoverable documents, pending/in-flight batches, and active storage operations. \`hasPendingPush\` remains separate Git state. Changes without an open document, such as board ordering or bulk operations, remain \`CommitBatcher\` entries.

**### Removal**

Remove \`MarkdownEditorStateStore\`, its props, instances, subscriptions, and tests. \`CardBodySaveStatus\` subscribes to the canonical card document and reads file-level dirty state instead of combining two sources. Rename generic per-file queries such as \`hasPendingActionFile\` where still needed for conflict handling.

**## Failure and lifecycle rules**

\- Switching section, tab, view, project, branch, or closing the app flushes editor drafts into persistence before batches.
\- Failed flushes/commits keep document and global persistence state dirty and retryable.
\- External changes renew a clean document's draft; for a dirty document, existing conflict/recovery behavior applies and local content is never silently replaced.
\- Board/list editing uses one draft; non-origin editors synchronize without echo writes.
\- Closing the last view of an invalid/conflicted action preserves its recoverable draft.
\- Reopening a fully saved, released document creates a fresh wrapper and history.

**## Implementation scope**

\- Replace ID-only Markdown binding snapshots/events with full documents plus optional section targets.
\- Preserve the exact outgoing target while flushing during Markdown editor/history-monitor switching.
\- Make card/action data sources use document drafts and current paths.
\- Add canonical registry and list/board membership APIs to \`OpenFilesService\`.
\- Add document/revision metadata and successful-save callbacks to \`CommitBatcher\` entries for writes and renames.
\- Align action draft save acknowledgement and project persistence aggregation with batch completion.
\- Update board/list components, active-card hooks, toolbar/diff consumers, cleanup handlers, and save status to use full documents.
\- Delete compound action Markdown ownership helpers once no internal history migration requires them.

**## Acceptance criteria**

\- Board/list views of one card receive the same \`CardOpenDocument\`, draft, dirty state, and events.
\- Editing any card body, action field, prompt, or phrase marks exactly one file-level document dirty.
\- Successful physical persistence clears only the saved revision; failures/newer edits remain dirty.
\- Prompt/phrase switching writes the outgoing section while the action remains one dirty document.
\- Invalid/conflicted actions remain recoverable and dirty without a queued batch.
\- Save indicators, flush guards, project/branch switching, and app close observe all dirty documents and pending batches.
\- No \`MarkdownEditorStateStore\`, editor-specific persisted dirty flag, public compound \`markdownDocumentId\`, or pre-persistence \`savedRevision\` remains.
\- Tests cover shared board/list editing, section routing, renewal/rename, close/reopen, invalid actions, external conflicts, failed saves, and edits during in-flight saves.
\- App lint and tests pass.

**## See also**

\- \[\[J-017]]
\- \[\[J-018]]
\- \[\[B-052]]
\- \[\[B-068]]
\- \[\[B-071]]
