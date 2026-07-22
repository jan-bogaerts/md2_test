---
author: 
id: F_4
internalId: ec632bef-01bd-4e16-ae1a-402cbc2d26c4
title: Show augmentation restrictions in image info
status: new
owner: 
affects:
policy:
after: 
agents:
  - design/activity/card__ec632bef-01bd-4e16-ae1a-402cbc2d26c4.json#conversation=agent-d6b85e43-2083-4376-8f99-5b8badffa7f7
---

## Goal

Show Observation augmentation restrictions in image/video info, with an independent visibility toggle. Extract the `Info shown` submenu from `search_result_item_menu.jsx` to keep the main context menu focused.

## User needs

- Reviewers can inspect restriction assignments without opening **Training augmentations** and hide them to reduce overlay noise.
- Developers have the `Info shown` state and markup isolated in a dedicated component.

## Current state

- `SearchResultItemMenu` owns and subscribes to `imageInfoVisibility`, and renders **Info shown** inline.
- `activeViewService.imageInfoVisibility` has session-only `uri`, `indexDetails`, and `value` fields (all default `true`) with corresponding `showImageInfo*` setters.
- `SearchResultInfoOverlay` and `SearchResultTooltip` subscribe to visibility changes, pass visibility to `buildInfoLines`, clear cached lines, and reload.
- `buildInfoLines` conditionally renders URI, index details, and YAML-dumped `item.value`; it does not render `avoidAugmentations`.
- The overlay and tooltip consider the surface empty when those three fields are disabled.
- F-134 stores Observation restrictions in `avoidAugmentations`; F-136 centralized their labels in `src/services/filtering/augmentation_restriction_options.js` for filtering and **Training augmentations**.

## Requirements

### Visibility service

Extend `activeViewService.imageInfoVisibility` with `augmentationRestrictions`:

- Default it to `true`.
- Preserve partial updates and include it in the no-change comparison.
- Add a `showImageInfoAugmentationRestrictions` getter/setter that changes only this field.
- Keep visibility session-only; do not add config persistence.

### Menu extraction

Create `src/main_window/images/search_result_info_menu_item.jsx`:

- Own the `imageInfoVisibility` React state and subscription to `activeViewService.events.imageInfoVisibility`.
- Render the top-level **Info shown** `MenuItem`, nested `Menu`, check marks, and toggle handlers for **URI**, **Index details**, **Value**, and **Augmentation restrictions**.
- Retain the existing MUI primitives and submenu placement.
- Keep its API small: the parent must not pass individual visibility fields or handlers.

In `search_result_item_menu.jsx`:

- Remove the inline **Info shown** state, effect, handlers, and markup.
- Render `SearchResultInfoMenuItem` in the same location below **Show/Hide image info**.
- Preserve parent-owned context-menu close behavior.

### Info output

Update `buildInfoLines` to add a line only when `visibility.augmentationRestrictions !== false` and `item.avoidAugmentations` contains at least one tag.

- Map tags to display labels with `AUGMENTATION_RESTRICTION_GROUPS`; use the raw tag for unknown values.
- Use a compact format such as `Augmentation restrictions: No shift left, No zoom out`.
- Never emit an empty restrictions line.

Update `SearchResultInfoOverlay` and `SearchResultTooltip` so `hasVisibleFields` includes `augmentationRestrictions`. Visibility changes must still clear cached lines and reload the displayed info.

## Acceptance criteria

- With image info shown and **Augmentation restrictions** enabled, an Observation with restriction tags gets a readable line in both overlay and tooltip; an Observation without tags gets no restrictions line.
- Toggling **Augmentation restrictions** updates its check mark and reloads both surfaces with the new visibility.
- If it is the only enabled field, items with tags show the restrictions rather than **No info selected**.
- If every field is disabled, both surfaces still show **No info selected**.
- Existing URI, index-details, and value toggles retain their behavior.
- **Info shown** behaves as before in `search_result_item_menu.jsx`, apart from the new option.
- Tests cover visibility defaults/setters, restriction formatting and visibility in `buildInfoLines`, `SearchResultInfoMenuItem`, and overlay/tooltip empty-field checks.

## Out of scope

- Editing restrictions from the overlay or tooltip.
- Changing restriction storage or **Training augmentations** behavior.
- Persisting visibility across restarts.

## Implementation note

Import the existing restriction labels; do not duplicate them. The extracted component owns all submenu state and behavior, not only its JSX.
