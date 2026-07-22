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

## Problem

F-047 provides one Electron action runner with correct chaining, `okButNotAfter`, and cancellation semantics. An audit against `design/architecture/initial description/writings/running_actions.md` found 17 contract gaps. Each finding below gives its location, required behavior, and fix.

Decisions:

- Every command run writes history, even without a commit line (finding 12 is a bug).
- Conversation continuation uses an extended Electron runner start request, not a fake action ID (finding 4).
- Free-prompt panel chat uses the builtin `md2.custom-prompt` action (finding 4b).

## Findings

### 1. No card `currentAction` state (missing, high)

**Contract:** While a card action runs, keep an in-memory `currentAction`; disable every action entry point for that card; clear it on completion, failure, cancellation, or `okButNotAfter`; publish execution-state changes; persist nothing.

**Gap:** No `currentAction` exists in `app/src`. `app/src/components/actions/action_entry_points.tsx` never disables entry points. The running dot in `app/src/components/card_view/project_card_view.tsx` (~line 57, `isAgentRunning`) uses only `card.agentConversations`, so command runs are invisible and concurrent runs are possible.

**Fix:** From the shared `onActionExecution` stream, maintain `cardPath -> executionId` in one renderer subscription (for example, `AgentIntegration` or a dedicated service). Set it on the root `execution`/`running` event and clear it on `completed`, `failed`, `cancelled`, or `okButNotAfter`. Finding 2 supplies the card path via event `context`. Disable all `ActionEntryPoints` variants plus popup `Run`/`Schedule` for that card. The map naturally starts empty. React-test disabling during a run and re-enabling after every terminal status.

### 2. Execution events omit `context` (missing, high)

**Contract:** Every event identifies the root action, current action, context, phase, and execution status.

**Gap:** `emit(execution, action, phase, status, details)` in `desktop/src/actions/action_runner_service.js` (~line 484) emits `actionId`, `executionId`, `phase`, `rootActionId`, `status`, and details, but no `context`. `ActionExecutionEvent` in `app/src/data/action_run_types.ts` also lacks it.

**Fix:** Emit `context: execution.context`, add `context: ActionContext` to `ActionExecutionEvent`, and extend event assertions in `desktop/src/actions/action_runner_service.test.mjs`. Finding 1 depends on this.

### 3. Web mode leaves execution controls enabled (missing, high)

**Contract:** Without an Electron backend, disable run, cancel, and live-input controls with a clear explanation; definitions remain editable.

**Gap:** Popup `Run` (`app/src/components/actions/action_popup.tsx`) is always enabled. Without a bridge, `runElectronAction` throws `Action execution requires Electron` (`app/src/services/electron_action_runner.ts`, ~line 59), producing only a failed run log after the click. Panel `Start`/`Continue`/`Send` in `app/src/components/agents/agent_conversation_list.tsx` behave likewise.

**Fix:** Expose `hasExecutionBackend` from bridge presence (`getElectronActionBridge() !== null`); remote-control browser sessions count because they proxy to Electron. Disable `Run`, `Cancel`, `Schedule`, and panel `Start`/`Continue`/`Send`, with text such as “Action execution requires the Electron desktop app.” React-test the disabled state and explanation.

### 4. Conversation continuation bypasses the runner (missing, high)

**Contract/F-047:** Manual, `onState`, scheduled, related, and continuation runs all use the Electron runner. Panel continuation “starts or resumes a linked Electron agent run.”

**Gap:** `continueConversation` in `app/src/services/agent_conversation_service.ts` (lines 160–186) loads the source conversation, builds the prompt via `buildContinuePrompt`, and calls raw bridge `storage.startAgentConversation` (`desktop/src/shell/local_bridge_dispatch.js`, lines 181–185). It has no execution ID, chain semantics, shared events, or card execution state.

**Fix (decided):**

- Add an optional continuation field such as `runInput.continueFrom: <conversation reference>` to the runner start request and `ALLOWED_RUN_INPUT_FIELDS` in `desktop/src/actions/action_runner_service.js`.
- In Electron, load that conversation, resolve `nativeSessionId`, and build the resume command with `buildResumeAgentCommand`; move the renderer's `buildContinuePrompt` fallback to Electron. Run it as a normal agent action, including events, history, and cancellation.
- Persist the originating action ID in conversation logs. Panel `Continue` calls `runElectronAction` with it; older conversations fall back to `md2.custom-prompt`.
- Remove renderer-side prompt building. Keep the `startAgentConversation` bridge only for the search-regexp agent unless finding 4b removes its last other caller.

### 4b. Free-prompt panel chat bypasses the runner (missing, medium)

**Gap:** `startAgentConversation` in `app/src/services/agent_integration.ts` (line 163), used by panel `Start`, launches a raw agent process.

**Fix (decided):** Route `Start` through `runElectronAction(BUILTIN_CUSTOM_PROMPT, fileContext, { extraPrompt: prompt })`. The runner already resolves `md2.custom-prompt` from `shared/action_definitions.mjs`. Remove `AgentIntegration.startAgentConversation` and `AgentConversationService.startConversation` when unused.

### 5. Run popup ignores executable availability (missing, medium)

**Contract:** At startup, detect supported Codex and Claude executables; clearly explain and disable unavailable choices.

**Gap:** Only the action editor consults `agentCapabilitiesService` (`app/src/components/actions/action_agent_capability_fields.tsx`). The run popup's agent select (`app/src/components/actions/action_agent_form.tsx`) lists `mergeAgentProfiles(...)` without availability, so spawn is the first failure. Detection is lazy on first editor open, not at startup.

**Fix:** When an Electron bridge exists, load availability once during app bootstrap into `agentCapabilitiesService`. Reuse `useAgentCapabilities` in `use_action_popup_controller.ts`/`ActionAgentForm`; disable unavailable profiles with the editor's `name — error` label and disable `Run` when one is selected.

### 6. Conversation panel lacks popup parity (missing, medium)

**Contract:** The panel shows the active execution and popup-equivalent history; command actions show status and logs without agent chat controls.

**Gap:** `app/src/components/agents/agent_conversation_list.tsx` renders only persisted agent conversations: no action history or command executions.

**Fix:** For the active card/file, render (a) live execution from the shared stream/store (finding 1), including command status and logs, and (b) history through the popup's `loadActionHistory` path. Reuse `app/src/services/action_history.ts` and `app/src/components/actions/action_run_history.tsx`. Show chat inputs only for running agent executions.

### 7. `sendActionInput` is unused; stdin takes the legacy path (incorrect, medium)

**Gap:** `sendActionInput(executionId, input)` is wired through `desktop/src/shell/local_bridge_dispatch.js` (lines 236–240), preload, and remote control, but production never calls it. The panel uses `sendAgentInput(conversation.id, input)`, which works only because agent run IDs equal conversation IDs in `desktop/src/actions/agent_runner_service.js`.

**Fix:** For running action executions, call `sendActionInput(executionId, input)` using the live execution tracked by findings 1/6. Retain `sendAgentInput` only while a non-runner path exists; after findings 4/4b, remove it so stdin has one path.

### 8. Popup state is local, not shared (incorrect, high)

**Contract:** Popup and panel share live execution state, which remains visible.

**Gap:** `use_action_popup_controller.ts` stores `runStatus`, `runResult`, and `executionId` in component state populated by the `runElectronAction` promise. Closing and reopening mid-run resets to `idle`, removes `Cancel`, and cannot reattach.

**Fix:** Feed a renderer-wide store from one `onActionExecution` subscription: `executionId -> { rootActionId, context, status, logs }`. On mount, the popup selects by action ID + context and reattaches to a running or recent execution. Replace `runElectronAction`'s per-call subscription with store reads; its current subscribe-before-`startAction` buffering race then disappears. This is the same store used by findings 1/6.

### 9. Popup has no live log streaming (incorrect, medium)

**Gap:** `runElectronAction` in `app/src/services/electron_action_runner.ts` (lines 82–84) ignores `running` events, although the runner emits stdout/stderr chunks through `emit(..., 'running', { stdout, stderr, ... })`. Output appears only when a phase ends, and no UI consumes command streams live.

**Fix:** Append running-event chunks to the finding 8 store and render them in popup and panel logs while running. Treat the final phase event as authoritative complete output.

### 10. Global indicator double-counts agent actions (incorrect, medium)

**Gap:** In `app/src/services/agent_integration.ts`, `handleActionExecutionEvent` (lines 295–330) adds `Action <rootActionId>` on the root `execution`/`running` event, then `agentConversationService.observeRunEvent` adds another entry for each embedded `agentEvent` of type `started`. Agent actions show twice; commands once.

**Fix:** Keep one execution-event entry for both action types. Do not call `observeRunEvent` for agent events with `event.executionId`; retain it for non-runner paths until findings 4/4b remove them.

### 11. Popup labels cancelled phases as failed (incorrect, low)

**Gap:** `logFromEvent` in `app/src/services/electron_action_runner.ts` (line 44) maps every non-`completed` status to `failed`.

**Fix:** Preserve `cancelled`, widen `ActionRunLogEntry.status`, and update rendering/colors.

### 12. Commands without a commit line have no history (bug, medium — decided)

**Gap:** `appendCommandHistory` in `desktop/src/actions/action_runner_service.js` (lines 468–476) returns when `extractCommitMetadata` finds no `[branch hash]` line, violating history keyed by action ID and context.

**Fix:** Always append `{ command, completedAt, output, prompt: '', status }`; add `commit` only when found, matching the nearby agent-entry shape. Test that a command without a commit line writes history.

### 13. Remarkable builtin is hardcoded in the runner (design, medium)

**Gap:** `REMARKABLE_CONVERT_ACTION` in `desktop/src/actions/action_runner_service.js` (lines 15–36) is special-cased by `loadRootAction`. `loadRequestAction` in `desktop/src/shell/local_bridge_dispatch.js` (lines 48–65) does not know it, so history loading for `md2.convert-remarkable-images-to-text` throws `Unknown action`. The runner therefore writes inaccessible history (latent because no UI currently opens it).

**Fix:** Define it in `shared/action_definitions.mjs` beside `BUILTIN_CUSTOM_PROMPT`; include it in the registry/actions returned by `validateActionDefinitionGraph`, with equivalent duplicate-ID protection. Remove the runner special case so `loadRootAction` and `loadRequestAction` resolve it normally. Keep it out of entry-point listings through renderer `appliesTo`/`builtin` handling; verify `actionsForContext` filtering.

### 14. Desktop duplicates definition loading (design, medium)

**Gap:** `ActionRunnerService.loadRootAction` and `local_bridge_dispatch.loadRequestAction` both load files, call `loadActionDefinitions`, sanitize errors, and find IDs. They have drifted: builtin handling differs; dispatch needs renderer-supplied `actionsFolder`, while the runner owns it from `startProject`.

**Fix:** Extract one resolver, for example `desktop/src/actions/action_definition_resolver.js`, accepting `(localGitService, project, actionsFolder, profiles, actionId)`, and use it in both places. Use the runner-owned `actionsFolder` for history too; remove it from renderer `ActionRunHistoryRequest` to tighten the Electron boundary.

### 15. Commit parser is duplicated across processes (design, low)

**Gap:** `extractCommitMetadata`/`COMMIT_LINE_PATTERN` are duplicated in `desktop/src/actions/action_runner_service.js` (lines 10, 115–129) and `app/src/services/action_history.ts` (lines 11–38).

**Fix:** Move the parser and pattern to `shared/` (as with `shared/agent_profiles.mjs`) and import them in both processes; keep call-site input differences such as `project.rootPath`. If the renderer copy has no callers after findings 4/4b, delete it instead.

### 16. `okButNotAfter` lacks distinct presentation (design, low)

**Gap:** `statusColor` in `app/src/components/actions/action_popup_defaults.ts` (lines 64–70) falls through to `text.secondary`; only raw status text distinguishes it. Panel/conversation chips in `agent_conversation_list.tsx` do not support it.

**Fix:** Use `warning.main` and label it “Completed, after-actions failed” in popup status, history, and the finding 6 panel.

### 17. `onState` detection is renderer-only (design, documented limitation)

**Current behavior:** `triggerStateActions` in `app/src/services/agent_integration.ts` (lines 227–237) detects card changes and invokes `runElectronAction`. Execution is Electron-owned, satisfying acceptance, but external state changes trigger nothing unless the app is running with a loaded snapshot.

**Fix:** No contract code change. With explicit approval, document this in `running_actions.md` under “Application-level triggers.” Moving triggers to Electron is a separate feature requiring card-state diffing on `watchProject` events.

## Suggested fix order

1. **2:** Add event `context`; unblocks 1, 6, and 8.
2. **8, 9, 1:** Build one live-execution store for popup reattachment, live logs, and card `currentAction`.
3. **3, 5:** Add backend and executable gating.
4. **12, 11, 16:** Land independent correctness/presentation fixes.
5. **4, 4b, 7, 10:** Route continuation/free chat through the runner, then collapse stdin and indicator paths. Finding 10's `executionId` guard can land earlier.
6. **13, 14, 15:** Complete independent desktop refactors.
7. **17:** Documentation only; requires approval.

## Test plan

- **Desktop:** Every runner event includes `context`; commands without commit lines write history; continuation resolves the Electron resume command and rejects unknown conversation references; both runner and history loader resolve the Remarkable builtin; unit-test the shared resolver.
- **React:** Card entry points and popup controls disable while running and re-enable for every terminal status; reopening reattaches the popup; live chunks render; cancellation remains cancelled; web mode explains disabled controls; unavailable agents disable in the popup; each execution has one global indicator; the panel shows command status and popup-equivalent history; panel input uses `sendActionInput`.
- Run `npm run lint` and `npm run test` in `app/` and `desktop/`, plus `npm run typecheck` in `app/`.

## See also

- `design/architecture/initial description/writings/running_actions.md`
- `design/feature_descriptions/F_047_running_actions_and_agents.md`
- `design/feature_descriptions/B_050_action_execution_still_has_two_orchestrators.md`
