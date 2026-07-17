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

Show Observation `avoidAugmentations` in image/video info overlays and tooltips, with a separate `Info shown` toggle for augmentation restrictions.

Also extract the existing `Info shown` submenu from `search_result_item_menu.jsx` into its own component so the main context menu stays focused.

## Changes

- Add `augmentationRestrictions` to `activeViewService.imageInfoVisibility`, defaulting to `true`.
- Add `showImageInfoAugmentationRestrictions` getter/setter while preserving partial-update behavior.
- Create `src/main_window/images/search_result_info_menu_item.jsx` to own:
  - `imageInfoVisibility` React state
  - the active view subscription
  - the `Info shown` submenu
  - toggle items for URI, Index details, Value, and Augmentation restrictions
- Update `search_result_item_menu.jsx` to render the new component below `Show/Hide image info`.
- Update `buildInfoLines` to show a compact line such as:
  - `Augmentation restrictions: No shift left, No zoom out`
- Reuse `AUGMENTATION_RESTRICTION_GROUPS` for labels; unknown tags may fall back to raw tags.
- Update overlay and tooltip empty-state checks so augmentation restrictions count as visible info.

## Acceptance Criteria

- Items with `avoidAugmentations` show readable restriction labels when the toggle is enabled.
- Items without restrictions do not show an empty restrictions line.
- Toggling `Augmentation restrictions` updates the menu check mark and reloads overlay/tooltip info.
- If only augmentation restrictions are enabled and the item has tags, the info surface does not show `No info selected`.
- If all fields are disabled, the existing `No info selected` behavior remains.
- URI, Index details, and Value toggles keep their current behavior.
- The `Info shown` submenu behaves the same as before, except for the new option.
- Tests cover visibility defaults/setters, restriction line output, the new menu component, and overlay/tooltip empty-field checks.

## Out of Scope

- Editing augmentation restrictions from overlay or tooltip.
- Changing restriction storage.
- Persisting visibility settings across restarts.
- Changing the `Training augmentations` menu.
