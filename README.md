# Hopcast Android SDK

Hopcast adds **peer-assisted content delivery** to your Android app: nearby devices exchange
your app's content directly, device-to-device, instead of re-downloading it from your
servers. Two complementary flows are available:

- **Online mode** — fully automatic. Your app tells Hopcast which contents each device
  *has* and which contents it *needs*; the Hopcast cloud matches nearby devices and the
  SDK performs the transfer in the background. No transfer UI to build.
- **Offline mode** — manual, AirDrop-like, and works **without any internet
  connection**. Your app drives discovery, connection and file sending through the SDK
  and gets callbacks for every step.

Both flows share one device identity, one cache directory and one set of listeners, and
can run side by side (`HYBRID` mode). Every exchange is reported to your Hopcast
dashboard, including whether it used the online or offline flow.

**This repository contains:**

| Path | Content |
|---|---|
| `maven/` | A static Maven repository holding the SDK (`fr.hopcast:sdk-hopcast-android`): compiled AAR, POM, Gradle module metadata, checksums |
| `CHANGELOG.md` | Release notes |

Requirements: **minSdk 21** (Android 5.0), compileSdk 34 recommended, Google Play
Services on the device (the device-to-device transport is Google Nearby Connections).

Looking for iOS? See [Hopcast/hopcast-sdk-ios](https://github.com/Hopcast/hopcast-sdk-ios).

---

## 1. Installation

The SDK is served straight from this repository — no account or credentials needed.

### 1.1 Declare the repository

**`settings.gradle.kts`** (Kotlin DSL, recommended):

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven("https://raw.githubusercontent.com/Hopcast/hopcast-sdk-android/main/maven")
    }
}
```

**`settings.gradle`** (Groovy DSL):

```groovy
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://raw.githubusercontent.com/Hopcast/hopcast-sdk-android/main/maven' }
    }
}
```

**Legacy projects** that still declare repositories in the root `build.gradle`:

```groovy
allprojects {
    repositories {
        google()
        mavenCentral()
        maven { url 'https://raw.githubusercontent.com/Hopcast/hopcast-sdk-android/main/maven' }
    }
}
```

### 1.2 Add the dependency

**`app/build.gradle.kts`**:

```kotlin
android {
    compileOptions {
        // Required: the SDK uses java.time / CompletableFuture back-ported to API 21+.
        isCoreLibraryDesugaringEnabled = true
        sourceCompatibility = JavaVersion.VERSION_1_8
        targetCompatibility = JavaVersion.VERSION_1_8
    }
    packaging {
        resources {
            // Duplicate metadata files shipped by the MQTT client's transitive dependencies.
            excludes += setOf(
                "META-INF/INDEX.LIST", "META-INF/io.netty.versions.properties",
                "META-INF/DEPENDENCIES", "META-INF/LICENSE", "META-INF/LICENSE.txt",
                "META-INF/NOTICE", "META-INF/NOTICE.txt",
            )
            pickFirsts += setOf("META-INF/native-image/**")
        }
    }
}

dependencies {
    implementation("fr.hopcast:sdk-hopcast-android:0.5.2")
    coreLibraryDesugaring("com.android.tools:desugar_jdk_libs:2.0.4")

    // Online / hybrid mode only — see the note below.
    implementation("com.google.firebase:firebase-messaging:23.4.1")
}
```

**`app/build.gradle`** (Groovy):

```groovy
android {
    compileOptions {
        coreLibraryDesugaringEnabled true
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    packaging {
        resources {
            excludes += ['META-INF/INDEX.LIST', 'META-INF/io.netty.versions.properties',
                         'META-INF/DEPENDENCIES', 'META-INF/LICENSE', 'META-INF/LICENSE.txt',
                         'META-INF/NOTICE', 'META-INF/NOTICE.txt']
            pickFirsts += ['META-INF/native-image/**']
        }
    }
}

dependencies {
    implementation 'fr.hopcast:sdk-hopcast-android:0.5.2'
    coreLibraryDesugaring 'com.android.tools:desugar_jdk_libs:2.0.4'

    // Online / hybrid mode only — see the note below.
    implementation 'com.google.firebase:firebase-messaging:23.4.1'
}
```

Notes:

- **Core library desugaring is mandatory**, whatever your minSdk: without it the build stops
  with *"Dependency 'fr.hopcast:sdk-hopcast-android:…' requires core library desugaring to
  be enabled"*.
- **Firebase Cloud Messaging is not pulled in by the SDK.** It is only used by the online
  mode, as a wake-up channel when Android has suspended your app. Offline-only apps don't
  need it. Apps using `ONLINE_ONLY` or `HYBRID` add `com.google.firebase:firebase-messaging`
  themselves (plus the usual `google-services` plugin and `google-services.json`). Without
  it the online mode still works whenever your app is running — it just can't be woken up.
- **R8 / ProGuard** — see [section 1.3](#13-r8--proguard-minified-release-builds).

### 1.3 R8 / ProGuard (minified release builds)

The AAR ships consumer rules that R8 applies to your app automatically. Since **0.5.2**
they cover, with R8 **full mode** (the AGP 8+ default):

- the SDK's public API, its manifest-referenced services and its listener interfaces;
- the MQTT client stack (HiveMQ, Netty, JCTools), which resolves some members by name at
  runtime — its members are kept, unused classes are still removed;
- the `-dontwarn` entries for optional classes those libraries reference (OkHttp's
  Conscrypt / BouncyCastle / OpenJSSE TLS providers, Netty's compression codecs, logging
  back-ends…), so a minified build does not stop on *"Missing class …"*.

The device-to-device wire protocol no longer depends on reflection at all, so it cannot be
altered by your R8 configuration: a debug (unminified) build and a release (minified)
build of your app exchange files with each other normally.

What this does **not** cover: rules your own code needs, and keep rules for dependencies
you add yourself (e.g. `firebase-messaging`, which ships its own). If you write custom
rules, don't `-dontshrink` / `-keep` the whole SDK — it is not necessary.

To verify a release build, call `HopcastSDK.runProtocolSelfCheck()` (no build or
provisioning needed): `passed` must be `true`, and the `fingerprint` must be identical in
your debug and release variants.

All SDK entry points live on the `fr.hopcast.sdk.HopcastSDK` object.

---

## Getting your SDK key

Your SDK key (`pk_live_…`) is obtained from your Hopcast account management console at
**https://console.hopcast.io**. It is designed to be embedded in your app and can be
revoked at any time from the console.

---

## 2. Integration guide

### 2.1 Runtime permissions

The SDK's manifest (merged automatically into yours) declares the Bluetooth, Wi-Fi,
location and nearby-devices permissions Nearby Connections needs. Request the runtime ones
before activating a mode:

```kotlin
val permissions = HopcastSDK.getRequiredPermissions().toTypedArray()
ActivityCompat.requestPermissions(activity, permissions, REQUEST_CODE)
```

### 2.2 Build the SDK (once per launch)

```kotlin
HopcastSDK.Builder(
    deviceId = "Alice's Pixel",                       // human-readable display name
    activity = this,                                  // or any Context
    lifecycleOwner = this,                            // or null to manage it with release()
    cacheDirectory = File(filesDir, "hopcast")        // where received files land
)
    .setMode(Mode.HYBRID)                             // OFFLINE_ONLY | ONLINE_ONLY | HYBRID
    .setServiceId("hopcast")                          // devices only see peers with the same id
    .setAutoAcceptConnections(false)                  // offline flow: ask the user first
    .setAutoAcceptTransfers(false)                    // offline flow: ask the user first
    .setOnlineConfig(OnlineConfig())                  // required for ONLINE_ONLY / HYBRID
    .build()
```

Notes:

- `cacheDirectory` is both where received files are written and where the online mode
  looks up files by content id when this device serves content. **The file name is the
  content id.**
- `Mode` decides which flows your app can activate at runtime; `HYBRID` allows both.
- `serviceId` defaults to your package name. Use the same value in your Android and iOS
  apps so they can discover each other.
- Auto-accept only affects the *manual* offline flow. Online-mode transfers are always
  automatic.

### 2.3 Enroll the device (once)

Call `provision` **after `build()`**, after your user's first sign-in, while the device is
online. This is a one-time operation: the resulting device identity is stored encrypted
(Android Keystore) and survives app restarts and offline periods.

```kotlin
HopcastSDK.provision(sdkKey = "pk_live_…", userId = user.id) { result ->
    result.onSuccess { /* The device is enrolled; the SDK is now operational. */ }
          .onFailure { /* Show a retry path — the SDK stays inert until provisioning succeeds. */ }
}
```

Need to enroll before building the SDK (e.g. right after login, elsewhere in your app)? Use
the overload taking a `Context`: `HopcastSDK.provision(context, sdkKey, userId) { … }`.

**The SDK is inert until provisioned**: every other API safely no-ops and logs a warning.
Check `HopcastSDK.isProvisioned` to decide whether to call `provision` again (calling it
when already provisioned succeeds immediately, without any network call). Failures are
`HopcastIdentityException.Server` (HTTP error), `HopcastIdentityException.InvalidResponse`
or an `IOException` (network).

`HopcastSDK.clearIdentity()` drops the identity (account switching, tests); re-provision
afterwards.

### 2.4 Register listeners

```kotlin
HopcastSDK.setDeviceDiscoveryListener(discoveryListener)
HopcastSDK.setConnectionListener(connectionListener, tag = "app")
HopcastSDK.setTransferListener(transferListener, tag = "app")
HopcastSDK.setOnlineEventListener(onlineListener, tag = "app")
```

Listeners are registered under a `tag` so different components can observe
independently (`remove…Listener(tag)` to unregister); callbacks are delivered on the main
thread. Most listener methods have a default empty implementation — override only what
you need.

### 2.5 Online mode — automatic, cloud-orchestrated

```kotlin
HopcastSDK.enableOnlineMode()
HopcastSDK.disableOnlineMode()
```

`OnlineEventListener.onCloudReady(deviceUuid)` fires when the device is connected and
available for orchestration; `onCloudDisconnected()` when it is not (the SDK reconnects
automatically).

**Declare inventory** — which contents this device holds, so it can serve them. Call this
after your own downloads complete, and on app start for contents already on disk:

```kotlin
// infraType = how the content was obtained: WIFI, CELLULAR, D2D or STORED
HopcastSDK.reportDownloadedFile(InfraType.WIFI, listOf("episode-1044"))

// When you delete a content from your cache:
HopcastSDK.reportRemovedFile(listOf("episode-1044"))
```

Contents received through Hopcast itself are declared automatically.

**Request content** — file a *need*:

```kotlin
val needId = HopcastSDK.declareNeed("episode-1044", ttlSeconds = 300)
// Later, if the need is obsolete:
needId?.let { HopcastSDK.cancelNeed(it) }
```

If a nearby device has the content, Hopcast triggers the transfer automatically: the
receiver gets `onFilesToBeDownloaded(fileIds)`, then regular `TransferListener` events, and
the file lands in `cacheDirectory` named after its content id. The sending side has
nothing to do. `onCloudError(reason)` reports orchestration failures.

Orchestrated deliveries coexist with your own offline transfers — see
[2.8 Concurrency](#28-concurrency-what-the-sdk-handles-for-you).

**Push wake-ups (recommended).** With `firebase-messaging` on your classpath:

- If Hopcast is your app's **only** FCM consumer, keep `OnlineConfig(isSecondaryFcm = false)`
  and declare the SDK's service in your manifest:

  ```xml
  <service
      android:name="fr.hopcast.sdk.internal.online.firebase.HopcastFirebaseMessagingService"
      android:exported="false">
      <intent-filter>
          <action android:name="com.google.firebase.MESSAGING_EVENT" />
      </intent-filter>
  </service>
  ```

- If your app already has its own `FirebaseMessagingService`, use
  `OnlineConfig(isSecondaryFcm = true)` and forward from it:

  ```kotlin
  override fun onMessageReceived(message: RemoteMessage) {
      HopcastSDK.handleFcmMessage(message.data)   // ignores messages that aren't Hopcast's
  }
  override fun onNewToken(token: String) {
      HopcastSDK.handleFcmToken(token)
  }
  ```

### 2.6 Offline mode — manual, works with no internet

```kotlin
HopcastSDK.enableOfflineMode()      // start advertising + discovery
HopcastSDK.disableOfflineMode()     // stop everything, disconnect
```

**Discovery** — the listener always receives the full current snapshot:

```kotlin
object : DeviceDiscoveryListener {
    override fun onNeighborsChanged(neighbors: List<Neighbor>) { /* neighbor.endpointId, neighbor.displayName */ }
}
```

**Connection:**

```kotlin
HopcastSDK.inviteNeighbors(listOf(neighbor.endpointId))

// Receiver side (ConnectionListener), unless auto-accept is on:
override fun onIncomingConnectionRequest(neighbor: Neighbor) {
    HopcastSDK.acceptInvitation(neighbor.endpointId)   // or rejectInvitation
}
override fun onConnectionStateChanged(state: ConnectionState, neighbor: Neighbor) { }
```

**Send files** to one or more neighbors:

```kotlin
val transferId = HopcastSDK.sendFiles(listOf(endpointId), listOf(file1, file2))
```

If a single target peer is already connected, the transfer starts right away and the call
returns its transfer id. Otherwise (several peers, or a peer that isn't connected yet) the
SDK queues a *bulk send*: it connects to each peer in turn, sends, and reports
`TransferEvent.BulkSendStarted` / `BulkSendFinished` (the call then returns the bulk id).

**Sending several files one call at a time (e.g. photo after photo).** A new connection
costs the Nearby handshake *plus* the upgrade from Bluetooth to a Wi-Fi Direct / Wi-Fi
Aware link, which can take **13–16 s between devices joined to different Wi-Fi networks**.
To pay it only once, keep the connection open across sends:

```kotlin
HopcastSDK.Builder(…)
    .setAutoDisconnectAfterTransfer(false)   // keep the link after each send
    .build()

HopcastSDK.inviteNeighbors(listOf(endpointId))           // or let the first sendFiles connect
// … once ConnectionState.CONNECTION_ESTABLISHED (or HopcastSDK.connectedPeers contains it):
photos.forEach { HopcastSDK.sendFiles(listOf(endpointId), listOf(it)) }   // reuse the link
// When done with that peer:
HopcastSDK.disconnect(endpointId)
```

- `setAutoDisconnectAfterTransfer(false)` applies to both paths (connected peer and bulk
  send). Without the call, a send to an already-connected peer keeps the link, and a bulk
  send drops each peer's link after its transfer (so another device can reach it).
  `setAutoDisconnectAfterTransfer(true)` drops the link after every send. The
  `sendFiles(endpointIds, files, metadata, disconnectAfterTransfer)` overload overrides it
  for one call.
- Sending a list of files in **one** `sendFiles` call is a single transfer over a single
  connection — the simplest way to send a batch.
- A device holds **one** device-to-device link at a time. Keeping a link open to peer A
  means a different device B cannot connect to it meanwhile; the SDK frees the link itself
  when *you* send to another peer (after transfers on it finish), but call `disconnect` when
  you no longer need it.

**Known limitation — mobile hotspot.** When a device has its Wi-Fi hotspot (tethering)
turned on, Android does not let Nearby Connections create the Wi-Fi Direct / Aware data
link (Nearby reports `OUT_OF_RESOURCE` internally): transfers still work but stay on
Bluetooth, i.e. much slower. This is an OS constraint that no SDK can work around; ask
users to turn the hotspot off for large transfers. `HopcastSDK.isWifiEnabled()` /
`setOnWifiStateChangedListener` cover the related "Wi-Fi off" case.

**Receive files** (unless auto-accept is on):

```kotlin
override fun onTransferInvitation(invitation: TransferInvitation) {
    HopcastSDK.acceptTransfer(invitation.transferId)   // or rejectTransfer
}
```

Either side can abort with `HopcastSDK.cancelTransfer(transferId)`.

### 2.7 Observing transfers (both modes)

```kotlin
object : TransferListener {
    override fun onTransferStateChanged(transferId: String, isReceiver: Boolean,
                                        state: TransferState, peer: Neighbor) { }
    override fun onTransferEvent(event: TransferEvent) {
        when (event) {
            is TransferEvent.TransferProgress -> { /* event.bytesTransferred / event.totalBytes */ }
            is TransferEvent.FileCompleted    -> { /* event.localPath on the receiver */ }
            is TransferEvent.TransferFailed   -> { /* event.reason */ }
            else -> Unit
        }
    }
}
```

Exchange reporting to your Hopcast dashboard is automatic in both modes. Reports are
queued durably on the device and delivered when connectivity allows; you have nothing to
implement.

### 2.8 Concurrency: what the SDK handles for you

Your app, the online orchestrator and the optional auto-sync can all want the same peer at
the same time. Since 0.5.2 the SDK arbitrates:

- **No duplicate connections.** `inviteNeighbors` (and every internal connection request)
  is ignored for a peer that is already connected or already connecting; Nearby's "already
  connected" answer is treated as success, never as a rejection.
- **One orchestrated delivery per peer at a time.** A cloud-orchestrated send to a peer
  that is busy (a transfer running with it, or another send queued for it) waits in the
  SDK's queue and starts on the same open link when it is free. Its transfer id is
  assigned immediately; `TransferListener` events follow when it actually starts.
- **A link in use is never torn down by the SDK.** Automatic disconnects (bulk sends,
  `setAutoDisconnectAfterTransfer(true)`) are skipped while any other transfer — incoming
  or outgoing, yours or orchestrated — still uses the link, or while a queued send is about
  to reuse it. Freeing the link for another peer waits for running transfers to finish.
- Your own `sendFiles` calls to an already-connected peer are not delayed: several of them
  may run in parallel on the same link.

What you still own: `disconnect(endpointId)` cuts the link immediately, including transfers
in progress on it. Check first with the read-only state:

```kotlin
HopcastSDK.connectedPeers                       // List<Neighbor> — established links
HopcastSDK.activeTransfers                      // List<ActiveTransfer> — running (and queued orchestrated) transfers
HopcastSDK.isTransferInProgress(endpointId)     // anything running / queued with that peer?

if (!HopcastSDK.isTransferInProgress(endpointId)) HopcastSDK.disconnect(endpointId)
```

`ActiveTransfer` carries `transferId`, `isReceiver`, `peer` and `isQueued`. These are
snapshots; use the listeners to react to changes.

### 2.9 Lifecycle

```kotlin
HopcastSDK.release()    // tears everything down; rebuild with the Builder to use again
```

For transfers that must outlive your activities, the SDK also provides an optional
foreground service, `fr.hopcast.sdk.service.HopcastService` (declare it in your manifest
with `android:foregroundServiceType="dataSync|connectedDevice"`). Start it with an intent
whose action is one of `ServiceActions.ENABLE_ONLINE`, `DISABLE_ONLINE`, `ENABLE_OFFLINE`,
`DISABLE_OFFLINE` to toggle the matching mode from the service.

---

## 3. API quick reference

| Area | API |
|---|---|
| Identity | `provision(sdkKey, userId, callback)`, `provision(context, sdkKey, userId, callback)`, `isProvisioned`, `clearIdentity()`, `deviceUuid` |
| Setup | `Builder(deviceId, context, lifecycleOwner, cacheDirectory)` → `setMode`, `setServiceId`, `setAutoAcceptConnections`, `setAutoAcceptTransfers`, `setAutoDisconnectAfterTransfer`, `setOnlineConfig`, `build()` |
| Modes | `enableOnlineMode()`, `disableOnlineMode()`, `enableOfflineMode()`, `disableOfflineMode()`, `mode` |
| Online | `reportDownloadedFile(infraType, contentIds)`, `reportRemovedFile(contentIds)`, `declareNeed(contentId, ttlSeconds)`, `cancelNeed(needId)`, `handleFcmToken(token)`, `handleFcmMessage(data)` |
| Offline | `startSharing(listener)`, `stopSharing()`, `inviteNeighbors(endpointIds)`, `acceptInvitation(endpointId)`, `rejectInvitation(endpointId)`, `disconnect(endpointId)`, `sendFiles(endpointIds, files, metadata)`, `sendFiles(endpointIds, files, metadata, disconnectAfterTransfer)`, `acceptTransfer(transferId)`, `rejectTransfer(transferId)`, `cancelTransfer(transferId)` |
| State (read-only) | `connectedPeers`, `activeTransfers`, `isTransferInProgress(endpointId)` |
| Listeners | `setDeviceDiscoveryListener`, `setConnectionListener`, `setTransferListener`, `setOnlineEventListener`, `setTransferHandoffListener` (+ matching `remove…` calls) |
| Misc | `getRequiredPermissions()`, `isWifiEnabled()`, `runProtocolSelfCheck()`, `release()` |

---

## 4. Troubleshooting

- **Build fails: "requires core library desugaring"** — enable it (section 1.2).
- **Build fails: "N files found with path 'META-INF/INDEX.LIST'"** — add the `packaging`
  excludes from section 1.2.
- **Gradle can't find `fr.hopcast:sdk-hopcast-android`** — the `maven(...)` repository must
  be declared where your project resolves dependencies (`settings.gradle(.kts)` with
  `dependencyResolutionManagement`, or `allprojects` in legacy projects).
- **Devices don't see each other** — both apps must use the same `serviceId`, have the
  runtime permissions granted, Bluetooth + Wi-Fi + location enabled. Transfers require
  physical devices with Google Play Services.
- **Online mode never reaches `onCloudReady` (0.5.1)** — 0.5.1's POM missed the WebSocket
  module of the MQTT client (`io.netty:netty-codec-http`). Fixed in 0.5.2; with 0.5.1, add
  `implementation("io.netty:netty-codec-http:4.1.99.Final")` to your app.
- **Release build: transfers never complete / `ClassCastException … LinkedTreeMap` /
  `Can't find '[toLeakAwareBuffer]'` (0.5.1)** — 0.5.1 was not robust to your app's R8
  full-mode pass. Upgrade to 0.5.2 (no app-side rule needed); `runProtocolSelfCheck()`
  confirms it.
- **Transfers between two devices are slow to start, every time** — each new connection
  pays a Wi-Fi upgrade (up to ~15 s across different Wi-Fi networks). Keep the link open
  across sends (section 2.6). If one device has its **mobile hotspot** on, transfers stay
  on Bluetooth (OS limitation).
- **Every call logs "not provisioned"** — `provision` has not succeeded on this device yet.
  Check `isProvisioned` and your SDK key.
- **`declareNeed` returns null** — the SDK is not provisioned, or the mode does not include
  the online flow (or no `OnlineConfig` was set).
- **Online transfers don't start** — both devices must have online mode enabled, be within
  range of each other, and the serving device must hold the content (declared via
  `reportDownloadedFile`, file present in `cacheDirectory` under the content id).

## Support

contact@hopcast.io — © Hopcast. See [LICENSE](LICENSE).
