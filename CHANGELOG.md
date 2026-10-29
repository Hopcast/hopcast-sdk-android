# Changelog

## 0.5.2 — 2026-10-29

Fixes for issues reported on Pixel 8a (Android 17) / Redmi (Android 15) with AGP 8.13 and
R8 full mode.

- **Online mode works out of the box**: the POM now declares `io.netty:netty-codec-http`
  (the MQTT-over-WebSocket transport, optional in the MQTT client). In 0.5.1 the `wss://`
  connection failed and `onCloudReady` never fired.
- **Minified apps**: the device-to-device protocol no longer uses reflection, so it is
  immune to the app's R8 pass (fixes `ClassCastException: LinkedTreeMap cannot be cast to
  …`, and transfers never completing). Wire format unchanged and identical between debug
  and release builds; 0.5.2 interoperates with 0.5.1 debug builds.
- Consumer rules now cover HiveMQ / Netty / JCTools (fixes `Can't find
  '[toLeakAwareBuffer]'`) and the OkHttp / Netty optional classes (`-dontwarn`); no rule
  needed in the app.
- New `HopcastSDK.runProtocolSelfCheck()` to verify a release build.
- **Connection reuse**: `setAutoDisconnectAfterTransfer(false)` now also keeps the link
  after bulk sends, so consecutive sends to the same peer skip the connection and Wi-Fi
  upgrade (13–16 s across networks). Without the call, behaviour is unchanged.
- **Concurrency**: duplicate connection requests are ignored and "already connected" is
  treated as success; cloud-orchestrated sends are serialized per peer; the SDK never tears
  down a link another transfer (or a queued send) still uses.
- New read-only state: `HopcastSDK.connectedPeers`, `activeTransfers`
  (`ActiveTransfer`), `isTransferInProgress(endpointId)`.
- `HopcastService` now handles `ServiceActions.ENABLE_ONLINE` / `DISABLE_ONLINE`.
- `androidx.lifecycle:lifecycle-runtime` is exposed as an `api` dependency
  (`Builder` takes a `LifecycleOwner`): apps without AppCompat compile.
- Documented: mobile hotspot on → Wi-Fi data link unavailable, transfers stay on
  Bluetooth (OS limitation).

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
