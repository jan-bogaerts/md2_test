---
author: 
id: F_4
internalId: ec632bef-01bd-4e16-ae1a-402cbc2d26c4
title: Show augmentation restrictions in image info
status: new
owner: 
affects:
policy:
after: f1150d6f-4ad8-454f-afa2-9e386d645b46
agents:
worktree: 1
---

## Goal

Show Observation augmentation restrictions in image/video info behind their own visibility toggle. Extract **Info shown** from `search_result_item_menu.jsx` to keep the context menu focused.

## User needs

- Reviewers: inspect restrictions without opening **Training augmentations**, or hide them to reduce noise.
- Developers: isolate **Info shown** state and markup.

## Current state

- `SearchResultItemMenu` owns/subscribes to `imageInfoVisibility` and renders **Info shown** inline.
- Session-only `activeViewService.imageInfoVisibility` has `uri`, `indexDetails`, and `value`; each defaults to `true` and has a `showImageInfo*` setter.
- The overlay and tooltip subscribe to visibility, pass it to `buildInfoLines`, clear cached lines, reload, and treat all three fields disabled as empty.
- `buildInfoLines` conditionally emits URI, index details, and YAML-dumped `item.value`, not `avoidAugmentations`.
- F-134 stores restrictions in `avoidAugmentations`; F-136 centralized their filtering/**Training augmentations** labels in `src/services/filtering/augmentation_restriction_options.js`.

## Requirements

### Visibility service

Add `augmentationRestrictions: true` to `activeViewService.imageInfoVisibility`, preserving partial updates and including it in no-change checks. Add a `showImageInfoAugmentationRestrictions` getter/setter that changes only this field. Keep it session-only; add no config persistence.

### Menu extraction

Create `src/main_window/images/search_result_info_menu_item.jsx`:

- Own the `imageInfoVisibility` React state and `activeViewService.events.imageInfoVisibility` subscription.
- Render **Info shown** and its nested `Menu`, check marks, and toggles for **URI**, **Index details**, **Value**, and **Augmentation restrictions**.
- Keep the MUI primitives and submenu placement. Accept no individual visibility fields or handlers from the parent.

In `search_result_item_menu.jsx`, replace the inline state, effect, handlers, and markup with `SearchResultInfoMenuItem` in the same location below **Show/Hide image info**. Keep context-menu closing parent-owned.

### Info output

In `buildInfoLines`, emit restrictions only if `visibility.augmentationRestrictions !== false` and `item.avoidAugmentations` has tags. Map them through `AUGMENTATION_RESTRICTION_GROUPS`, using raw unknown tags. Format as `Augmentation restrictions: No shift left, No zoom out`; never emit an empty line.

Include `augmentationRestrictions` in the overlay and tooltip `hasVisibleFields` checks. Visibility changes must still clear cached lines and reload both surfaces.

## Acceptance criteria

- With image info shown and restrictions enabled, tagged Observations get a readable line in both surfaces; untagged ones get none.
- Toggling restrictions updates its check mark and reloads both surfaces with the new visibility.
- With restrictions as the only enabled field, tagged items show them instead of **No info selected**.
- With every field disabled, both surfaces show **No info selected**.
- Existing URI, index-details, and value toggles retain their behavior; **Info shown** is otherwise unchanged except for the new option.
- Tests cover visibility defaults/setters, `buildInfoLines` restriction formatting/visibility, `SearchResultInfoMenuItem`, and overlay/tooltip empty-field checks.

## Out of scope

- Editing restrictions from the overlay or tooltip.
- Changing restriction storage or **Training augmentations** behavior.
- Persisting visibility across restarts.

## Implementation note

Import, rather than duplicate, the restriction labels. The extracted component owns all submenu state and behavior, not just JSX.
