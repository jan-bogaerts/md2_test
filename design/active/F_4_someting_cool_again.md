---
author: 
id: F_4
internalId: ec632bef-01bd-4e16-ae1a-402cbc2d26c4
title: Show augmentation restrictions in image info
status: new
owner: 
affects:
agents:
  - .md2-agent-logs/design_active_F_4_someting_cool_again.md_agent-2f9fe48d-1609-447e-8eeb-36b915cc833f.json
policy:
after: 
---
## Goal

Let users show or hide Observation augmentation restrictions in the image/video info overlay and tooltip. Extract the `Info shown` submenu from `search_result_item_menu.jsx` into its own component so the context menu stays focused.

## User Stories

1. As a reviewer, I want augmentation restrictions visible in image/video info so I can inspect assignments without opening the training augmentations menu.
2. As a reviewer, I want to toggle restriction info independently so I can reduce overlay noise.
3. As a developer, I want `Info shown` state and markup isolated so `search_result_item_menu.jsx` remains focused on the main context menu.

## Current State

- `SearchResultItemMenu` owns `imageInfoVisibility`, subscribes to `activeViewService.events.imageInfoVisibility`, and renders `Info shown` inline.
- `activeViewService.imageInfoVisibility` contains `uri`, `indexDetails`, and `value`, all defaulting to `true`.
- `SearchResultInfoOverlay` and `SearchResultTooltip` pass visibility to `buildInfoLines`.
- `buildInfoLines` renders URI, index details, and `item.value`; it does not show `avoidAugmentations`.
- F-134 stores restrictions on Observations as `avoidAugmentations`; F-136 moved reusable labels to `src/services/filtering/augmentation_restriction_options.js`.

## Implementation

- Add `augmentationRestrictions: true` to `activeViewService.imageInfoVisibility`.
- Preserve partial updates, include the new field in no-change comparisons, and add `showImageInfoAugmentationRestrictions`.
- Keep this as session state; do not persist it.
- Create `src/main_window/images/search_result_info_menu_item.jsx`.
- Move `Info shown` state, subscription, submenu markup, check marks, and toggle handlers into the new component.
- Include `URI`, `Index details`, `Value`, and `Augmentation restrictions` menu items.
- Use the same MUI menu primitives and submenu placement as the current inline implementation.
- Update `search_result_item_menu.jsx` to render `SearchResultInfoMenuItem` below `Show/Hide image info` while preserving parent close behavior.
- Update `buildInfoLines` to show `Augmentation restrictions: No shift left, No zoom out` when `visibility.augmentationRestrictions !== false` and `item.avoidAugmentations` has values.
- Use `AUGMENTATION_RESTRICTION_GROUPS` for labels; unknown tags may fall back to raw tags.
- Do not render an empty restrictions line.
- Update overlay and tooltip empty-field checks to include `augmentationRestrictions`, and keep visibility changes clearing cached lines/reloading displayed info.

## Acceptance Criteria

- Items with `avoidAugmentations` show a readable restrictions line when image info and `Augmentation restrictions` are enabled.
- Items without restrictions show no empty restrictions line.
- Toggling `Augmentation restrictions` updates the check mark and reloads overlay/tooltip info.
- If only `Augmentation restrictions` is enabled and an item has tags, overlay/tooltip do not show `No info selected`.
- If all fields are disabled, overlay/tooltip still show `No info selected`.
- Existing URI, Index details, and Value toggles keep their behavior.
- The `Info shown` submenu behaves the same except for the new option.
- Tests cover visibility defaults/setters, `buildInfoLines`, `SearchResultInfoMenuItem`, and overlay/tooltip empty-field checks.

## Out of Scope

- Editing restrictions from overlay or tooltip.
- Changing how tags are stored.
- Persisting visibility settings across restarts.
- Changing the `Training augmentations` menu.

## Notes

- Reuse existing restriction labels instead of duplicating them.
- The new component should own submenu state completely; this is an extraction, not just JSX.
