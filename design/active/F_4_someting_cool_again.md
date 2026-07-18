---
author: 
id: F_4
internalId: ec632bef-01bd-4e16-ae1a-402cbc2d26c4
title: Show augmentation restrictions in image info
status: new
owner: 
affects:
agents:
  - design/logs/conversation__card__active_f_4_someting_cool_again__agent_3729e6c2_d417_4f99_aa3c_36739e0de9e0.json
  - design/logs/conversation__card__active_f_4_someting_cool_again__agent_e3cb0b71_ed66_4296_9e90_9bd27e80bb98.json
policy:
after: 
---
\## Goal

Let users show or hide Observation augmentation restrictions in the image/video info overlay and tooltip, and move the \`Info shown\` submenu out of \`search\_result\_item\_menu.jsx\` so the menu stays manageable.



\## User Stories

1\. As a reviewer, I want augmentation restrictions visible in image/video info, so that I can inspect restriction assignments without opening the training augmentations menu.

2\. As a reviewer, I want to toggle augmentation restriction info independently, so that I can reduce overlay noise when it is not needed.

3\. As a developer, I want the \`Info shown\` menu state and submenu markup isolated in its own component, so that \`search\_result\_item\_menu.jsx\` remains focused on the main context menu.



\## Current state

\- \`SearchResultItemMenu\` owns \`imageInfoVisibility\` state, subscribes to \`activeViewService.events.imageInfoVisibility\`, and renders the \`Info shown\` submenu inline.

\- \`activeViewService.imageInfoVisibility\` currently contains \`uri\`, \`indexDetails\`, and \`value\`, all defaulting to \`true\`. It exposes \`showImageInfoUri\`, \`showImageInfoIndexDetails\`, and \`showImageInfoValue\` setters.

\- \`SearchResultInfoOverlay\` and \`SearchResultTooltip\` listen for \`imageInfoVisibility\` changes and pass the visibility object to \`buildInfoLines\`.

\- \`buildInfoLines\` renders URI, index details, and YAML-dumped \`item.value\` depending on visibility. It does not currently display \`avoidAugmentations\`.

\- Both overlay and tooltip treat the info surface as empty when only \`uri\`, \`indexDetails\`, and \`value\` are disabled.

\- F-134 stores augmentation restrictions on Observations as \`avoidAugmentations\`. F-136 moved the restriction labels to \`src/services/filtering/augmentation\_restriction\_options.js\` for reuse by filtering and the training augmentations menu.



\## implementation details

\- Extend \`activeViewService.imageInfoVisibility\` with an \`augmentationRestrictions\` boolean.

&#x20; \- Default should be \`true\`, matching the current default-visible behavior for URI, index details, and value.

&#x20; \- The setter must preserve the existing partial-update behavior and include \`augmentationRestrictions\` in the no-change comparison.

&#x20; \- Add \`showImageInfoAugmentationRestrictions\` getter/setter that updates only this field.

&#x20; \- This remains session state like the existing image info visibility; do not add config persistence unless a separate feature asks for it.

\- Create \`src/main\_window/images/search\_result\_info\_menu\_item.jsx\`.

&#x20; \- It owns the \`imageInfoVisibility\` React state and the subscription to \`activeViewService.events.imageInfoVisibility\`.

&#x20; \- It renders the top-level \`Info shown\` \`MenuItem\`, its nested \`Menu\`, check marks, and all toggle handlers.

&#x20; \- Include menu items for \`URI\`, \`Index details\`, \`Value\`, and \`Augmentation restrictions\`.

&#x20; \- Use the same MUI menu primitives and submenu placement currently used by \`search\_result\_item\_menu.jsx\`.

&#x20; \- Keep the component API small; the parent should not pass individual visibility fields or toggle handlers.

\- Update \`search\_result\_item\_menu.jsx\`.

&#x20; \- Remove the inline \`Info shown\` state, effect, handlers, and submenu markup.

&#x20; \- Render the new \`SearchResultInfoMenuItem\` in the same location below \`Show/Hide image info\`.

&#x20; \- Keep parent-owned close behavior for the whole context menu unchanged.

\- Update \`buildInfoLines\`.

&#x20; \- Include augmentation restriction lines when \`visibility.augmentationRestrictions !\=\= false\` and the item has one or more \`avoidAugmentations\` tags.

&#x20; \- Reuse \`AUGMENTATION\_RESTRICTION\_GROUPS\` to map tags to user-facing labels. Unknown tags may fall back to the raw tag.

&#x20; \- Use a compact output format, for example \`Augmentation restrictions: No shift left, No zoom out\`.

&#x20; \- Do not add a line when the item has no restrictions.

\- Update \`SearchResultInfoOverlay\` and \`SearchResultTooltip\`.

&#x20; \- Their \`hasVisibleFields\` checks must include \`augmentationRestrictions\`.

&#x20; \- Visibility changes must continue to clear cached lines and reload the displayed info.



\## Acceptance Criteria

\- Given an Observation has \`avoidAugmentations\`, when image info is shown and \`Augmentation restrictions\` is enabled, then the overlay and tooltip include a readable augmentation restrictions line.

\- Given an Observation has no \`avoidAugmentations\`, when \`Augmentation restrictions\` is enabled, then no empty restrictions line is shown.

\- Given the user opens \`Info shown\`, when they toggle \`Augmentation restrictions\`, then the check mark updates and both overlay and tooltip reload according to the new visibility state.

\- Given only \`Augmentation restrictions\` is enabled, when an item has restriction tags, then the overlay/tooltip do not show \`No info selected\`.

\- Given all info fields are disabled, when the overlay or tooltip is rendered, then it still shows \`No info selected\`.

\- Given existing code toggles URI, Index details, or Value, when this feature is added, then those toggles keep their current behavior.

\- Given \`search\_result\_item\_menu.jsx\` is rendered, when the \`Info shown\` menu is opened, then the submenu behavior is unchanged from the user's perspective except for the new augmentation restrictions option.

\- Tests cover \`activeViewService\` visibility defaults/setters, \`buildInfoLines\` restriction output and visibility behavior, the new \`SearchResultInfoMenuItem\`, and updated empty-field checks in overlay/tooltip.



\## Out of Scope

\- Editing augmentation restrictions from the info overlay or tooltip.

\- Changing how restriction tags are stored.

\- Persisting image info visibility settings across app restarts.

\- Changing the existing \`Training augmentations\` menu behavior.



\## Further Notes

\- Prefer importing the existing restriction label definitions instead of duplicating tag labels.

\- The new component should own the submenu state completely; this is an extraction of behavior, not just JSX.