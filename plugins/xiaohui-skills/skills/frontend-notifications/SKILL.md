---
name: frontend-notifications
description: Use when adding, rebuilding, or standardizing a frontend notification, toast, snackbar, or global message system, especially when the project needs queue-based transient messages, loading-to-success or error updates, click-to-dismiss behavior, and either a Vue 3 plus Pinia or React implementation that mirrors a global show, update, close API.
---

# Frontend Notifications

## Overview

Build notifications as a small system, not scattered helpers.

Default to a queue-based global store with three operations:

- `show(text, type, duration?)`
- `update(id, payload)`
- `close(id)`

This skill ships Vue 3 + Pinia and React reference implementations that mirror a production pattern: push a loading message, keep its id, then update that same message to success or error.

## Output Contract

When implementing or refactoring frontend notifications:

- create or update a single source of truth for message state
- mount one global renderer near the app root
- expose stable APIs for `show`, `update`, and `close`
- support at least `loading`, `success`, `info`, and `error`
- support optional auto-dismiss durations
- support updating an existing loading message instead of stacking duplicate follow-up messages

## Workflow

1. Find the current notification entry points.
   Look for toast, snackbar, alert, message, notice, or ad hoc success/error state.
2. Decide whether to preserve the existing visual language.
   In an established app, keep the current placement, icon style, and timing unless the user asks to redesign it.
3. Centralize state first.
   Prefer one store or provider over per-component arrays.
4. Add a single renderer near the app root.
   The renderer should observe global state and own placement, transitions, and per-type visuals.
5. Replace multi-step async UX with an id-based update chain.
   Example: `const id = show('Saving...', 'loading')` then `update(id, { type: 'success', text: 'Saved', duration: 2000 })`.
6. Keep message lifetimes explicit.
   Use `duration` for transient notifications and omit it for persistent loading or manual-dismiss messages.
7. Verify the full interaction path.
   Check creation, update, click-to-dismiss, stacking order, and auto-close timing.

## Architecture Rules

- Keep the notification queue global.
- Do not scatter repeated `setTimeout` dismissal logic across pages.
- Prefer updating an existing loading message over showing a second success or error toast for the same action.
- Keep transport or business logic outside the renderer component.
- Let page or store code decide message copy; let the renderer decide presentation.
- In existing products, reuse current tokens, colors, and icon conventions unless the user requests a redesign.

## Vue 3 + Pinia Default

Use this default when the project is Vue 3 or already uses Pinia.

Implementation shape:

- store file like `src/store/message.ts`
- renderer component like `src/components/MessageList.vue`
- root mount in `App.vue` or a top-level layout

Reference files:

- See `references/vue-pattern.md` for the general model and migration guidance
- See `references/theme-tokens.md` for shared notification style tokens
- Copy from `assets/vue3-pinia/message.ts`
- Copy from `assets/vue3-pinia/MessageList.vue`
- Copy optional styles from `assets/shared/notification-theme.css`
- Copy mount example from `assets/vue3-pinia/App.vue`

## React Default

Use this default when the project is React.

Implementation shape:

- provider file like `src/components/notifications/NotificationsProvider.tsx`
- hook file like `src/components/notifications/useNotifications.ts`
- renderer mounted by the provider near the app root

Reference files:

- See `references/react-pattern.md` for the general model and usage guidance
- See `references/theme-tokens.md` for shared notification style tokens
- Copy from `assets/react/NotificationsProvider.tsx`
- Copy from `assets/react/useNotifications.ts`
- Copy optional styles from `assets/shared/notification-theme.css`
- Copy mount example from `assets/react/App.tsx`

## Message Semantics

- `loading`: usually no duration on first show
- `success`: usually update from loading and auto-close
- `error`: usually update from loading and auto-close a bit slower if the copy is longer
- `info`: short-lived, low-priority feedback

Use ids to model one user action across multiple states:

```ts
const id = messageStore.show('Uploading...', 'loading')
messageStore.update(id, { type: 'success', text: 'Upload complete', duration: 2000 })
```

## Common Mistakes

- Showing a new success toast instead of updating the existing loading toast
- Creating multiple local message arrays instead of one global queue
- Rendering notifications inside page content instead of a root-level fixed overlay
- Auto-closing loading messages before the async operation actually completes
- Rebuilding the same store and renderer logic differently in every project
- Recreating notification logic in every page instead of calling a hook or store API
- Treating modal alerts and transient notifications as the same component
- Hardcoding colors in every renderer instead of routing them through a shared theme layer

## Validation

Before claiming the notification system is complete:

- verify a message can be shown from any relevant page or store
- verify a loading message can be updated in place
- verify `duration` actually dismisses the message
- verify click-to-dismiss works
- verify stacking order and placement look correct on desktop and mobile
- verify the root renderer is mounted exactly once
