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
  - .md2-agent-logs/design_active_F_4_someting_cool_again.md_agent-67ad1e96-973f-4ba4-9223-819311e8e42f.json
  - design/logs/conversation__card__active_f_4_someting_cool_again__agent_3729e6c2_d417_4f99_aa3c_36739e0de9e0.json
policy:
after: 
---
## Goal

Show/hide Observation augmentation restrictions in image/video info overlays and tooltips. Extract the `Info shown` submenu from `search_result_item_menu.jsx`.

## Requirements

- Add `augmentationRestrictions: true` to session-only `activeViewService.imageInfoVisibility`, preserving partial updates and exposing `showImageInfoAugmentationRestrictions`.
- Create `search_result_info_menu_item.jsx` to own submenu state, subscriptions, check marks, and toggles for URI, index details, value, and restrictions.
- Replace the inline submenu in `search_result_item_menu.jsx` with `SearchResultInfoMenuItem`, preserving placement and parent-menu closing.
- Update `buildInfoLines` to show `Augmentation restrictions: ...` when enabled and `avoidAugmentations` exists. Reuse `AUGMENTATION_RESTRICTION_GROUPS`; raw values are acceptable for unknown tags.
- Update overlay and tooltip `hasVisibleFields` checks without changing reload behavior.

## Acceptance Criteria

- Enabled restrictions show readable labels; absent restrictions show no empty line.
- Toggling updates the check mark and reloads both overlay and tooltip.
- Restrictions alone count as visible; disabling all fields still shows `No info selected`.
- Existing URI, index, value, and submenu behavior is unchanged.
- Tests cover service defaults/setters, line formatting/visibility, the menu component, and empty-field checks.

## Out of Scope

- Editing stored restriction tags.
- Persisting visibility across restarts.
- Changing the `Training augmentations` menu.

Use existing restriction-label definitions; the new component fully owns submenu state.
