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

Let reviewers view and independently hide Observation augmentation restrictions in the image/video info overlay and tooltip, without opening the training augmentations menu. Extract the `Info shown` submenu into its own component so `search_result_item_menu.jsx` stays focused on the main context menu.

## Current state

- `SearchResultItemMenu` owns `imageInfoVisibility`, subscribes to `activeViewService.events.imageInfoVisibility`, and renders `Info shown` inline.
- `activeViewService.imageInfoVisibility` contains `uri`, `indexDetails`, and `value`, all defaulting to `true`, with corresponding `showImageInfo...` setters.
- `SearchResultInfoOverlay` and `SearchResultTooltip` subscribe to visibility changes, pass visibility to `buildInfoLines`, and consider the surface empty when those three fields are disabled.
- `buildInfoLines` conditionally renders URI, index details, and YAML-dumped `item.value`; it does not render `avoidAugmentations`.
- F-134 stores Observation restrictions in `avoidAugmentations`. F-136 moved their labels to `src/services/filtering/augmentation_restriction_options.js` for reuse by filtering and the training augmentations menu.

## Implementation

### Visibility state

Extend `activeViewService.imageInfoVisibility` with `augmentationRestrictions`:

- Default it to `true`, consistent with the existing fields.
- Preserve partial updates and include it in the no-change comparison.
- Add a `showImageInfoAugmentationRestrictions` getter/setter that updates only this field.
- Keep this as session state; do not add config persistence.

### Menu extraction

Create `src/main_window/images/search_result_info_menu_item.jsx`:

- Own the `imageInfoVisibility` React state and subscription to `activeViewService.events.imageInfoVisibility`.
- Render the top-level `Info shown` `MenuItem`, nested `Menu`, check marks, and toggle handlers for `URI`, `Index details`, `Value`, and `Augmentation restrictions`.
- Reuse the current MUI primitives and submenu placement.
- Keep its API small: the parent must not pass individual visibility fields or toggle handlers.

In `search_result_item_menu.jsx`:

- Remove the inline `Info shown` state, effect, handlers, and markup.
- Render `SearchResultInfoMenuItem` in the same position, below `Show/Hide image info`.
- Leave parent-owned context-menu closing behavior unchanged.

### Info output

Update `buildInfoLines` to add restrictions only when `visibility.augmentationRestrictions !== false` and `item.avoidAugmentations` contains at least one tag.

- Map tags to readable labels with `AUGMENTATION_RESTRICTION_GROUPS`; fall back to the raw tag when unknown.
- Use a compact format, such as `Augmentation restrictions: No shift left, No zoom out`.
- Add no line when the item has no restrictions.

Update `SearchResultInfoOverlay` and `SearchResultTooltip`:

- Include `augmentationRestrictions` in `hasVisibleFields`.
- Continue clearing cached lines and reloading displayed info after visibility changes.

## Acceptance criteria

- With restrictions present and the option enabled, both overlay and tooltip show a readable restrictions line.
- With no restrictions, neither surface shows an empty restrictions line.
- Toggling `Augmentation restrictions` updates its check mark and reloads both surfaces using the new visibility state.
- If it is the only enabled field and the item has restrictions, neither surface shows `No info selected`.
- If every field is disabled, both surfaces still show `No info selected`.
- Existing URI, Index details, and Value toggles retain their behavior.
- Opening `Info shown` from `search_result_item_menu.jsx` behaves as before, apart from the added option.
- Tests cover visibility defaults/setters in `activeViewService`, restriction output and visibility in `buildInfoLines`, the new `SearchResultInfoMenuItem`, and the updated overlay/tooltip empty-field checks.

## Out of scope

- Editing restrictions from the overlay or tooltip.
- Changing restriction tag storage.
- Persisting visibility across app restarts.
- Changing the existing `Training augmentations` menu.

## Notes

- Import existing restriction labels; do not duplicate them.
- The new component must own all submenu state—this is a behavior extraction, not only a JSX extraction.
