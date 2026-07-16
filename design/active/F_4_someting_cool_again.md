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
  - .md2-agent-logs/design_active_F_4_someting_cool_again.md_agent-482c00d7-ecda-40f9-ac88-e1bc719ee6aa.json
policy:
after: 
---
## Goal

Let users toggle Observation augmentation restrictions in the image/video info overlay and tooltip. Extract the `Info shown` submenu from `search_result_item_menu.jsx` into its own component.

## User Stories

1. Reviewers can inspect augmentation restrictions from image/video info.
2. Reviewers can toggle restriction info independently.
3. Developers can keep `Info shown` state and markup out of `search_result_item_menu.jsx`.

## Current State

- `SearchResultItemMenu` owns and renders `Info shown` inline.
- `activeViewService.imageInfoVisibility` has `uri`, `indexDetails`, and `value`, all defaulting to `true`.
- Overlay and tooltip pass visibility to `buildInfoLines`.
- `buildInfoLines` does not show Observation `avoidAugmentations`.
- Reusable labels live in `src/services/filtering/augmentation_restriction_options.js`.

## Implementation

- Add session-only `augmentationRestrictions: true` to `activeViewService.imageInfoVisibility`.
- Preserve partial updates, no-change checks, and add `showImageInfoAugmentationRestrictions`.
- Create `src/main_window/images/search_result_info_menu_item.jsx`.
- Move `Info shown` state, subscription, submenu, check marks, and toggles into it.
- Include `URI`, `Index details`, `Value`, and `Augmentation restrictions`.
- Render `SearchResultInfoMenuItem` below `Show/Hide image info` in `search_result_item_menu.jsx`, preserving parent close behavior.
- Update `buildInfoLines` to show labeled restrictions when enabled and `item.avoidAugmentations` has values.
- Use `AUGMENTATION_RESTRICTION_GROUPS`; unknown tags may use raw tags.
- Do not render empty restriction lines.
- Include `augmentationRestrictions` in overlay/tooltip empty-field checks and reload behavior.

## Acceptance Criteria

- Restrictions appear only when enabled and present.
- Toggling restrictions updates the check mark and overlay/tooltip info.
- Restriction-only visibility does not show `No info selected` when tags exist.
- Disabling all fields still shows `No info selected`.
- Existing URI, Index details, and Value behavior is unchanged.
- `Info shown` behaves the same, with the new option added.
- Tests cover defaults/setters, `buildInfoLines`, `SearchResultInfoMenuItem`, and empty-field checks.

## Out of Scope

- Editing restrictions from overlay/tooltip.
- Changing tag storage.
- Persisting visibility settings.
- Changing `Training augmentations`.

## Notes

- Reuse existing restriction labels.
- The new component owns submenu state completely.
