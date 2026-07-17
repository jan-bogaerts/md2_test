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
policy:
after: 
---
## Goal

Show Observation augmentation restrictions in the image/video info overlay and tooltip, with an independent visibility toggle. Extract the `Info shown` submenu from `search_result_item_menu.jsx` into its own component.ddkkk

## Requirements

* Add `augmentationRestrictions: true` to `activeViewService.imageInfoVisibility`.
* Preserve partial updates and no-change detection; add a `showImageInfoAugmentationRestrictions` getter/setter. Keep this session-only state.
* Create `src/main_window/images/search_result_info_menu_item.jsx` to own visibility state, event subscription, submenu rendering, check marks, and toggle handlers for URI, Index details, Value, and Augmentation restrictions.
* Replace the inline submenu in `search_result_item_menu.jsx` with `SearchResultInfoMenuItem`, preserving menu placement and close behavior.
* Update `buildInfoLines` to show, when enabled and present, a compact line such as `Augmentation restrictions: No shift left, No zoom out`. Map tags through `AUGMENTATION_RESTRICTION_GROUPS`, falling back to raw tags for unknown values.
* Include `augmentationRestrictions` in the overlay and tooltip `hasVisibleFields` checks. Continue clearing cached lines and reloading on visibility changes.

## Acceptance Criteria

* Restrictions appear in the overlay and tooltip only when enabled and tags exist; no empty line appears otherwise.
* Toggling the new option updates its check mark and reloads both displays.
* Restrictions alone count as visible info; disabling every field still shows `No info selected`.
* Existing URI, Index details, and Value behavior remains unchanged.
* Tests cover service defaults/setters, restriction formatting and visibility, the extracted menu item, and updated empty-state checks.

## Out of Scope

* Editing or changing storage of restriction tags.
* Persisting visibility across app restarts.
* Changing the existing `Training augmentations` menu.