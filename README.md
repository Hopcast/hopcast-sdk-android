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
    implementation("fr.hopcast:sdk-hopcast-android:0.5.1")
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
    implementation 'fr.hopcast:sdk-hopcast-android:0.5.1'
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
- **R8 / ProGuard**: the SDK ships its own consumer rules; you don't need to keep anything.
  If your minified release build fails with *"Missing class org.conscrypt…"* (optional TLS
  providers referenced by OkHttp), add to your `proguard-rules.pro`:

  ```
  -dontwarn org.bouncycastle.jsse.**
  -dontwarn org.conscrypt.**
  -dontwarn org.openjsse.**
  ```

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

**Send files** to one or more connected neighbors:

```kotlin
val transferId = HopcastSDK.sendFiles(listOf(endpointId), listOf(file1, file2))
```

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

### 2.8 Lifecycle

```kotlin
HopcastSDK.release()    // tears everything down; rebuild with the Builder to use again
```

For transfers that must outlive your activities, the SDK also provides an optional
foreground service, `fr.hopcast.sdk.service.HopcastService` (declare it in your manifest
with `android:foregroundServiceType="dataSync|connectedDevice"`).

---

## 3. API quick reference

| Area | API |
|---|---|
| Identity | `provision(sdkKey, userId, callback)`, `provision(context, sdkKey, userId, callback)`, `isProvisioned`, `clearIdentity()`, `deviceUuid` |
| Setup | `Builder(deviceId, context, lifecycleOwner, cacheDirectory)` → `setMode`, `setServiceId`, `setAutoAcceptConnections`, `setAutoAcceptTransfers`, `setOnlineConfig`, `build()` |
| Modes | `enableOnlineMode()`, `disableOnlineMode()`, `enableOfflineMode()`, `disableOfflineMode()`, `mode` |
| Online | `reportDownloadedFile(infraType, contentIds)`, `reportRemovedFile(contentIds)`, `declareNeed(contentId, ttlSeconds)`, `cancelNeed(needId)`, `handleFcmToken(token)`, `handleFcmMessage(data)` |
| Offline | `startSharing(listener)`, `stopSharing()`, `inviteNeighbors(endpointIds)`, `acceptInvitation(endpointId)`, `rejectInvitation(endpointId)`, `disconnect(endpointId)`, `sendFiles(endpointIds, files, metadata)`, `acceptTransfer(transferId)`, `rejectTransfer(transferId)`, `cancelTransfer(transferId)` |
| Listeners | `setDeviceDiscoveryListener`, `setConnectionListener`, `setTransferListener`, `setOnlineEventListener`, `setTransferHandoffListener` (+ matching `remove…` calls) |
| Misc | `getRequiredPermissions()`, `isWifiEnabled()`, `release()` |

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
- **Every call logs "not provisioned"** — `provision` has not succeeded on this device yet.
  Check `isProvisioned` and your SDK key.
- **`declareNeed` returns null** — the SDK is not provisioned, or the mode does not include
  the online flow (or no `OnlineConfig` was set).
- **Online transfers don't start** — both devices must have online mode enabled, be within
  range of each other, and the serving device must hold the content (declared via
  `reportDownloadedFile`, file present in `cacheDirectory` under the content id).

## Support

contact@hopcast.io — © Hopcast. See [LICENSE](LICENSE).
