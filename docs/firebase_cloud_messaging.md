# Firebase Cloud Messaging (FCM) integration plan (desktop)

## Context / goals
AyuGramDesktop is a fork of the AyuGram desktop Telegram client, focusing on customization and privacy features.

FCM is a cross-platform messaging system that allows a server to notify a client that new data is available to sync and to send notification messages for re-engagement. For instant messaging, it supports small payloads (up to 4096 bytes) for the notification/data body.

**Goal:** Add an optional push-notification wakeup layer to AyuGramDesktop using FCM (or an equivalent backend-triggered push) that can:
- wake the client to sync dialogs/messages
- update unread counters
- show OS-level notifications when appropriate

## Design constraints
- FCM client support varies by platform and build toolchain.
- Desktop builds (Windows/macOS/Linux) do not have the same “always-on Google Play Services” environment as Android.
- The desktop app can already deliver notifications while running; the new value is the ability to _wake_ the app and synchronize state.

## Proposed architecture
### 1) Backend push broker
Create a small backend service (or Cloud Function) that uses the FCM v1 API / Admin SDK to send **data messages** with minimal metadata:
- `type`: `new_message`, `reaction`, `mention`, `media_ready`, etc.
- `chat_id` / `dialog_id`
- optional: `message_id`, `preview` (short), `unread_count`

Token management:
- On desktop sign-in, register a device token with the backend.
- Store per-account device tokens.
- On token refresh, update the backend.

### 2) Client-side push bridge
Add a new module (conceptually):
- `Telegram/Push/` (or similar)
- responsibilities:
  - maintain FCM client session (where supported)
  - receive push payloads
  - deduplicate/pool them (don’t hammer Telegram servers for each message)
  - hand off to existing dialog/message sync pipeline
  - decide whether to show a notification based on foreground/background state

### 3) Build system
Add a CMake toggle for optional push support:
- `DESKTOP_APP_USE_FCM` (OFF by default)
- if enabled:
  - add third-party dependencies
  - add `Push` sources/headers
  - compile platform-specific bindings

### 4) Privacy / minimal payload
- Keep FCM payload metadata-only; the desktop client fetches full content via Telegram MTProto sync.
- Do not send full message text unless explicitly enabled by the user.
- Store tokens in OS secure storage (Keychain/DPAPI/Secret Service) where possible.

## Phase plan
**Phase 0:** baseline doc + toggle; no behavior change.
**Phase 1:** token registration + backend broker; client receives pushes but only logs them.
**Phase 2:** sync wakeups + OS notifications; shipping feature behind a user setting.

## Open questions
- Which desktop platforms are in scope (Windows/macOS/Linux)?
- What library/vendor will provide the desktop FCM client hooks?
- How should multi-account sessions route tokens and notifications?
