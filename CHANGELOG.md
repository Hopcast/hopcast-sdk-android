# Changelog

## 0.5.1 — 2026-10-14

First public binary distribution, as a static Maven repository (`maven/`).

- Online mode: cloud-orchestrated device-to-device delivery over MQTT (needs/has,
  automatic transfers, FCM wake-up), same protocol as the iOS SDK.
- Offline mode: manual AirDrop-like discovery, connection and file transfer on Google
  Nearby Connections, fully functional without internet. `HYBRID` mode runs both.
- Shared device identity: `provision` is idempotent, with a `Context` overload usable
  before `build()`; `clearIdentity()`; typed `HopcastIdentityException` failures.
- Exchange reporting for both modes, each record tagged `online` / `offline`; durable
  queue with backoff, flushed automatically.
- Tagged transfer-handoff listeners (`setTransferHandoffListener(listener, tag)`).
- Advertising / discovery resume when the app returns to the foreground.
- Hardened release build (R8); public API and service classes preserved for host apps.

Requires core library desugaring and, for online / hybrid mode, `firebase-messaging` in
the host app — see the README installation section.
