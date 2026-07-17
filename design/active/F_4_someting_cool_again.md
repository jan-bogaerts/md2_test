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

Show or hide Observation augmentation restrictions in the image/video info overlay and tooltip. Extract the `Info shown` submenu from `search_result_item_menu.jsx` into its own component.

## Requirements

- Add `augmentationRestrictions` to `activeViewService.imageInfoVisibility`, defaulting to `true`. Preserve partial updates, add `showImageInfoAugmentationRestrictions`, and keep this as session-only state.
- Create `src/main_window/images/search_result_info_menu_item.jsx` to own visibility state, event subscriptions, the `Info shown` submenu, check marks, and toggle handlers for URI, index details, value, and augmentation restrictions.
- Replace the inline submenu in `search_result_item_menu.jsx` with `SearchResultInfoMenuItem`, preserving its placement and parent menu-closing behavior.
- Update `buildInfoLines` to show a compact `Augmentation restrictions: ...` line when enabled and `avoidAugmentations` is present. Reuse `AUGMENTATION_RESTRICTION_GROUPS`; unknown tags may use their raw values.
- Update overlay and tooltip `hasVisibleFields` checks and retain their existing reload behavior when visibility changes.

## Acceptance Criteria

- Enabled restrictions appear with readable labels; absent restrictions produce no empty line.
- Toggling the new option updates its check mark and reloads both overlay and tooltip.
- Restrictions alone count as visible information; disabling every field still shows `No info selected`.
- Existing URI, index details, value, and submenu behavior remain unchanged.
- Tests cover service defaults/setters, restriction line formatting and visibility, the new menu component, and empty-field checks.

## Out of Scope

- Editing or changing stored restriction tags.
- Persisting visibility across app restarts.
- Changing the `Training augmentations` menu.

Use the existing restriction-label definitions rather than duplicating them. The new component should fully own the submenu state.
