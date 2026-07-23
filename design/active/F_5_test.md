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
policy:
after: 2f8cfca2-95da-464f-b14c-b5980a74d2d9
---

**## Goal**

Make one canonical \`OpenDocument\` own each open file’s in-memory draft and file-level dirty state. List and board views share it, preventing divergence in content, save state, renewal, and events. Dirty means the complete JSON or Markdown file still needs successful persistence; prompt, phrase, body, and editor-instance dirty flags do not exist.

**## Current problems**

- \`MarkdownEditorStateStore\` tracks an editor buffer while \`ActionService\` revisions and \`CommitBatcher\` track the same file through later save stages.
- Markdown data sources discard \`OpenDocument\` identity, retain string IDs, and reconstruct domain objects.
- Board and list card editors can hold different local buffers for one card.
- \`ActionService.savedRevision\` advances when \`DataService.persistActionFile()\` schedules a batch, before \`storage.commit()\` succeeds.
- \`CardBodySaveStatus\` masks the split by combining editor dirty state with \`CommitBatcher\` state.

**## Design**

**### Canonical open documents**

\`OpenFilesService\` owns a registry keyed by stable card/action identity and two membership sets: list-view and board-view documents. Both reference the same canonical \`OpenDocument\`. Opening a registered card in another view reuses it and does not open or activate a list tab; closing one view removes only that membership. Keep a document registered while either view owns it or it has unsaved/recoverable state. Close and project-switch flows must explicitly flush, discard, or retain that state.

Each document is a stable \`EventTarget\` exposing its full current domain object, draft, file-level dirty state, and private edit/save revisions. Domain renewal updates the stable object. Draft, dirty, saved, and renewal transitions emit document events; consumers read properties directly. Do not add \`getSnapshot()\` for a boolean.

**### Shared drafts**

Every edit updates the active document’s draft and marks the whole file dirty. Data sources receive and retain full documents; they never reconstruct ownership from paths or compound IDs. Both card editors read and subscribe to the same document, so edits propagate between bindings. Origin metadata prevents the originating editor from replacing its own state or losing cursor/history.

\`MarkdownEditor\` reports edits through the active document/data source but never declares a persisted document clean. Local transient editors without an \`OpenDocument\` may retain \`onDirtyChange\`; they are outside project-file save state.

**### Action sections**

An action has one \`ActionOpenDocument\` and one dirty file. Prompt and phrases are editor sections only:

\`\`\`ts
type ActionMarkdownSection =
  | { kind: 'prompt' }
  | { kind: 'phrase'; identity: string }
\`\`\`

The active target pairs the canonical action document with its section. Section identity routes outgoing text, external replacement, undo history, and phrase deletion; it never owns dirty state and is not encoded as a public namespaced \`markdownDocumentId\`. Card body uses one implicit \`body\` section.

History and echo suppression key by canonical document plus section, while history remains binding-owned. Closing a document discards its histories; deleting a phrase discards only that section.

**### Save ownership**

\`OpenDocument.dirty\` is authoritative:

1. A mutation increments the document’s private edit revision and marks it dirty.
2. Persistence scheduling captures the document, revision, and complete serialized file.
3. \`CommitBatcher\` maintains operational pending/in-flight entries per file.
4. Only successful \`storage.commit()\` acknowledges the captured revision.
5. A failed commit stays dirty; an older completion cannot clear newer edits.

Invalid action drafts remain dirty even when unschedulable. Once repaired, serialize and queue the complete current action. \`ActionService\` may retain draft, validation, conflict, and queue revisions, but saved revision means successful batch persistence—not handoff to \`CommitBatcher\`.

Path changes carry the same document and revision through the batch; a successful rename renews its path without replacing identity.

\`ProjectPersistenceService\` remains aggregate coordinator. Derive pending-save state from dirty/recoverable documents, pending/in-flight batches, and active storage operations. Keep \`hasPendingPush\` separate as Git state. Files changed without an open document (for example, board ordering or bulk operations) remain \`CommitBatcher\` entries.

**### Removal**

Remove \`MarkdownEditorStateStore\`, its props, instances, subscriptions, and tests. \`CardBodySaveStatus\` subscribes to the canonical card document and reads file-level dirty state; it no longer combines sources. Rename generic per-file batch queries such as \`hasPendingActionFile\` where still needed for conflict handling.

**## Failure and lifecycle rules**

- Switching section, tab, view, project, branch, or closing the app flushes editor drafts to persistence before batches.
- A failed flush or storage commit keeps the document and global persistence state dirty and retryable.
- External changes renew a clean document’s draft; against a dirty document, use existing conflict/recovery behavior and never silently replace local content.
- Simultaneous board/list editing shares one draft; non-origin editors synchronize without echo writes.
- Closing the last view of an invalid or conflicted action must retain its recoverable draft.
- Reopening a fully saved, released document creates a fresh wrapper and history.

**## Implementation scope**

- Replace ID-only Markdown binding snapshots/events with full documents plus optional section targets.
- Update Markdown editor and history-monitor switching to retain the exact outgoing target during flush.
- Make card/action data sources use document drafts and current document paths.
- Add the canonical registry and list/board membership APIs to \`OpenFilesService\`.
- Add document/revision metadata and successful-save callbacks to \`CommitBatcher\` entries for normal writes and renames.
- Align action draft-save acknowledgement and project-persistence aggregation with batch completion.
- Update board/list components, active-card hooks, toolbar/diff consumers, cleanup handlers, and save status to use full documents.
- Delete compound action Markdown document ownership helpers when no internal history migration still requires them.

**## Acceptance criteria**

- Board and list views of one card receive the same \`CardOpenDocument\`, draft content, dirty state, and events.
- Editing any card body, action field, prompt, or phrase marks exactly one file-level document dirty.
- Successful physical batch persistence clears only the saved revision; failures and newer edits remain dirty.
- Prompt/phrase switching always writes the outgoing section while the action remains one dirty document.
- Invalid/conflicted actions remain recoverable and dirty without a queued batch.
- Project save indicators, flush guards, project/branch switching, and app close observe all dirty documents and pending batches.
- No \`MarkdownEditorStateStore\`, editor-specific persisted dirty flag, public compound \`markdownDocumentId\`, or false pre-persistence \`savedRevision\` remains.
- Tests cover shared board/list editing, section routing, renewal/rename, close/reopen, invalid actions, external conflicts, failed saves, and edits during an in-flight save.
- App lint and tests pass.

**## See also**

- [[J-017]]
- [[J-018]]
- [[B-052]]
- [[B-068]]
- [[B-071]]
