# MagnetAd Android SDK — Publisher Integration Guide

A step-by-step guide to using the **MagnetAd** advertising package in **Android (Kotlin/Java)** projects to display **fullscreen (Interstitial)** and **Rewarded** ads.

> 🇮🇷 نسخهٔ فارسی این راهنما: [readme_fa.md](readme_fa.md)

---

## Contents

1. [Overview](#1-overview)
2. [Requirements](#2-requirements)
3. [Step 1 — Adding the package (AAR file)](#step-1--adding-the-package-aar-file)
4. [Step 2 — Manifest and ProGuard](#step-2--manifest-and-proguard)
5. [Step 3 — Getting your IDs (App Id and Placement Id)](#step-3--getting-your-ids)
6. [Step 4 — Initializing the SDK](#step-4--initializing-the-sdk)
7. [Step 5 — Loading and showing an ad](#step-5--loading-and-showing-an-ad)
8. [Step 6 — Events (AdListener) and the load result (AdLoadResult)](#step-6--events-adlistener-and-the-load-result-adloadresult)
9. [Step 7 — Video and rewarded ads](#step-7--video-and-rewarded-ads)
10. [Controlling video volume](#controlling-video-volume)
11. [Using several placements at once](#using-several-placements-at-once)
12. [Full API reference](#full-api-reference)
13. [Error codes](#error-codes)
14. [How the SDK behaves internally](#how-the-sdk-behaves-internally)
15. [Troubleshooting](#troubleshooting)
16. [Limitations](#limitations)

---

## 1. Overview

- Ad type: **fullscreen (Interstitial)** and **Rewarded**, with an **image or video** creative.
- Platform: **Android** (minimum API 21 / Android 5.0).
- Written in **Kotlin** with **Coroutines**; straightforward to use from Kotlin and fully usable from Java as well.
- Networking uses **OkHttp** and response parsing uses **kotlinx.serialization** (because distribution is via an AAR file, you add these dependencies to your `build.gradle` yourself — see [Step 1](#step-1--adding-the-package-aar-file)). Video playback uses Android's own `MediaPlayer`, so no extra playback library is added to your project.
- No dangerous permissions and no runtime permission dialog; only the normal `INTERNET`, `ACCESS_NETWORK_STATE` and `AD_ID` permissions are merged into your manifest.
- Every public method is **non-blocking**; network work happens in the background and results are delivered on the **main thread**: the load result through `requestAd`'s own callback (an `AdLoadResult`), and post-show events through `AdListener`.
- The usage pattern matches standard SDKs (AdMob / AppLovin): you create an ad object, listen for events, then load and show.

> **Important about the creative type:** you do not choose whether an ad is an image or a video. The server decides per placement and the SDK selects the right renderer automatically. Your code is identical either way.

---

## 2. Requirements

| Item | Value |
|------|-------|
| Minimum Android version | **API 21** (Android 5.0) |
| Your project's compileSdk | **34** or higher |
| Your project's Kotlin version | **1.9** or higher |
| Language | **Kotlin** (recommended) or Java |
| Java/JVM target | **11** |
| Build system | **Gradle** (Kotlin DSL or Groovy) |

> Your app's compileSdk does **not** have to be 36. The SDK's dependencies were chosen so it works from **compileSdk 34 upward**, so projects still on an older SDK are not forced to upgrade.

Permissions and settings the package merges into your app automatically (via Manifest Merger):

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="com.google.android.gms.permission.AD_ID" />
```

> You do not need to add these manually; the package's manifest merges them into your app's final manifest. None of them is a **dangerous** permission, so no permission dialog is ever shown to the user.

### About the `AD_ID` permission

For better ad targeting, the SDK adds the `com.google.android.gms.permission.AD_ID` permission to your manifest. What you need to know:

- If you publish on **Google Play**, you must declare Advertising ID usage in the Play Console's *Data safety* section (no such requirement applies when publishing to **Myket / Cafe Bazaar**).
- If your app targets **children**, or you simply do not want this permission in your app, you can remove it in your own manifest:

```xml
<uses-permission android:name="com.google.android.gms.permission.AD_ID"
    tools:node="remove" />
```

> With the permission removed, the SDK keeps working and ads are still shown; only targeting accuracy is reduced. To use `tools:node`, remember to add `xmlns:tools="http://schemas.android.com/tools"` to your `<manifest>` tag.

---

## Step 1 — Adding the package (AAR file)

MagnetAd is currently **not** published to any public Maven repository (Maven Central, JitPack, …). The only official distribution point is the GitHub repository below, and installation is done via the **AAR file**:

```
https://github.com/MagnetAds/magad-android-sdk
```

> This means `implementation("com.hasin:magnetad:…")` will not work, and you should not add any Maven repository for MagnetAd to `settings.gradle.kts`. Follow the steps below instead.

### 1-1) Getting the AAR file

Download the AAR from the repository above (the **Releases** section, or wherever the repository's `README` points to) and copy it into your project's `app/libs/` folder. Create the `libs` folder if it does not exist:

```
<your-project>/
└── app/
    └── libs/
        └── magnetad-1.8.0.aar
```

> The filename carries the version number. When upgrading, **delete** the old file and update the filename in `build.gradle.kts`; if both files stay in `libs`, the build fails with a duplicate class error.

### 1-2) Adding the dependency and its side dependencies

Because the AAR does not come from a Maven repository, there is no POM file alongside it — so **its side dependencies are not resolved automatically and must be added manually**. In your **app module's** `build.gradle.kts`:

```kotlin
dependencies {
    implementation(files("libs/magnetad-1.8.0.aar"))

    // Required by the SDK. Without these the project still compiles,
    // but crashes at runtime with NoClassDefFoundError:
    implementation("org.jetbrains.kotlin:kotlin-stdlib:2.0.21")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.8.1")
    implementation("com.google.android.gms:play-services-ads-identifier:18.1.0")
}
```

The Groovy equivalent (if you use `build.gradle`):

```groovy
dependencies {
    implementation files('libs/magnetad-1.8.0.aar')

    implementation 'org.jetbrains.kotlin:kotlin-stdlib:2.0.21'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1'
    implementation 'androidx.core:core-ktx:1.12.0'
    implementation 'com.squareup.okhttp3:okhttp:4.12.0'
    implementation 'com.squareup.okhttp3:logging-interceptor:4.12.0'
    implementation 'org.jetbrains.kotlinx:kotlinx-serialization-json:1.8.1'
    implementation 'com.google.android.gms:play-services-ads-identifier:18.1.0'
}
```

Important notes about these versions:

- Keep `androidx.core:core-ktx` at **1.12.0**. Newer versions require `compileSdk 36` and would force your project to upgrade.
- If your project already has any of these libraries (directly, or through another SDK), do **not** add it twice; Gradle picks the highest version itself. Just make sure the resolved version is not older than the ones above.
- `play-services-ads-identifier` is needed for ad targeting; if you leave it out, the SDK neither fails at build time nor crashes at runtime, but targeting accuracy is reduced.

### 1-3) Repositories for the side dependencies

The dependencies above come from public repositories, so your `settings.gradle.kts` needs them (they are usually already in your project):

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
}
```

> If `dl.google.com` is not reachable (regional blocking), add the `maven { url = uri("https://maven.myket.ir/") }` mirror **before** `google()`. Note this is only a mirror for public dependencies — MagnetAd itself is not published there.

---

## Step 2 — Manifest and ProGuard

### 2-1) Manifest (automatic)

The package merges the following into your app's final manifest automatically, and **you do not need to configure anything**:

- The `INTERNET`, `ACCESS_NETWORK_STATE` and `com.google.android.gms.permission.AD_ID` permissions (none of them dangerous — see [About the `AD_ID` permission](#about-the-ad_id-permission)).
- A `<queries>` element so installed stores (Cafe Bazaar / Myket / Google Play) can be detected on Android 11 and above.

That is all. The package's manifest has **no `<application>` tag**, so it overrides none of your app's settings.

> **Changed from earlier versions:** the package no longer adds `android:usesCleartextTraffic="true"` or a `networkSecurityConfig` to your manifest; all communication with the service is over `https`. Practical consequence for you: if your app has a custom `networkSecurityConfig`, there is no longer any Manifest Merger conflict and no need for `tools:replace`.

### 2-2) ProGuard / R8 (automatic)

The necessary rules are applied automatically through `consumer-rules.pro`, which is bundled inside the AAR itself; **you do not need to add any rules manually**. Those rules cover:

- Keeping the SDK's public classes (`MagnetAdManager`, `InterstitialAd`, `AdConfig`, `AdError`, `AdListener`, `AdLoadResult`, `AdLoadCallback`).
- `kotlinx.serialization` rules — without them, in a minified release build the server response would not be parseable and every request would fail with `INVALID_RESPONSE`.
- `-dontwarn` rules for OkHttp/Okio and the optional TLS providers (Conscrypt / BouncyCastle / OpenJSSE) — these are classes OkHttp uses only if they are present; without the rules, their absence would turn into a "missing class" error in your release build.

> If your release build fails with an R8 error about any of the packages above, the AAR's rules are not being applied; first make sure you picked up the AAR from an up-to-date version.

---

## Step 3 — Getting your IDs

You receive two identifiers from the Magnet publisher panel:

- **App Id** — used in `AdConfig` when calling `initialize`.
- **Placement Id** — the placement identifier; used in `getInterstitialAd`.

> These are strings, and **validating their format is the server's responsibility**.

> The placement type (regular or rewarded) is set when you create it in the panel, not in your code. Get a separate Placement Id for each type.

> Use the same IDs from the panel throughout development and release.

---

## Step 4 — Initializing the SDK

Initialize the SDK **once**, in your `Application` class. The call returns immediately and the handshake with the server happens in the background.

```kotlin
import android.app.Application
import android.util.Log
import com.hasin.magnetad.AdConfig
import com.hasin.magnetad.MagnetAdManager

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        MagnetAdManager.initialize(
            application = this,
            config = AdConfig(
                appId = "YOUR-APP-ID",
                debugMode = BuildConfig.DEBUG // SDK logs only in debug builds
            ),
            onInitComplete = { success ->
                // Optional; invoked on the main thread
                Log.d("MagnetAd", "init success = $success")
            }
        )
    }

    override fun onTrimMemory(level: Int) {
        super.onTrimMemory(level)
        MagnetAdManager.onTrimMemory(level) // helps release the image cache under memory pressure
    }
}
```

Then register your Application class in the manifest:

```xml
<application
    android:name=".MyApplication"
    ... >
```

> You may call `getInterstitialAd` even before the network handshake completes, but ad loading will fail until the handshake succeeds.

> **If `onInitComplete` reports `false`** (for example the user had no connectivity at launch), you can call `initialize` again later with the same parameters; the second call genuinely retries the handshake. Only a **successful** handshake causes later calls to be ignored, so retrying is harmless and you do not need to track the state yourself.

---

## Step 5 — Loading and showing an ad

Loading and showing are completely separate and explicit: you load with `requestAd`, receive a single-use `adId` on success, and pass that same ID to `show` whenever you want. The SDK never loads an ad on its own without an explicit call from you. Golden rule: **load early, keep the `adId`, and show at a natural break in your game or app.**

```kotlin
import android.os.Bundle
import android.util.Log
import androidx.activity.ComponentActivity
import com.hasin.magnetad.*

class MainActivity : ComponentActivity() {

    private var interstitialAd: InterstitialAd? = null
    private var cachedAdId: String? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 1) Create an ad object for a specific placement
        interstitialAd = MagnetAdManager.getInterstitialAd(placementId = "YOUR-PLACEMENT-ID")

        // 2) Listen to show-related events (everything after show)
        interstitialAd?.setAdListener(object : AdListener() {
            override fun onAdShown() { /* the ad appeared on screen */ }
            override fun onAdClicked() { /* the user tapped the ad */ }
            override fun onAdDismissed() { /* the ad was closed */ }

            override fun onAdFailedToShow(error: AdError) {
                Log.w("MagnetAd", "show failed: ${error.code}")
            }
        })

        // 3) Explicit load; the result arrives only through this callback
        interstitialAd?.requestAd(this) { result ->
            when (result) {
                is AdLoadResult.Success -> {
                    // The ad is ready; keep the adId and show it whenever you want
                    cachedAdId = result.adId
                    cachedAdId?.let { interstitialAd?.show(this@MainActivity, it) }
                }
                is AdLoadResult.Failure -> {
                    Log.w("MagnetAd", "load failed: ${result.error.code} — ${result.error.message}")
                }
            }
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        // 4) Release resources and cancel any in-flight load
        interstitialAd?.destroy()
        interstitialAd = null
    }
}
```

> **Hold on to the `InterstitialAd` instance.** Every `getInterstitialAd` call builds a **brand new** instance with an empty cache. If you call it again at show time, `show` fails with `AD_NOT_READY`, because the `adId` belongs to the instance that loaded it, not to the placement.

> To show another ad, call `requestAd` again and get a fresh `adId`. Each `adId` is valid for exactly one `show` — it is retired as soon as the ad is shown (or expires), so every `InterstitialAd` instance needs a new load after each show.

### The same code in Java

The SDK's public API is ready for Java consumption: `MagnetAdManager`'s methods are available as `static` (no `.INSTANCE` needed), default parameters have Java overloads, `AdListener` is an **abstract class** with empty implementations (so you override only the events you care about), and `AdLoadCallback` is a **functional interface**, so it works with a Java lambda.

```java
import android.app.Activity;
import android.os.Bundle;
import android.util.Log;
import androidx.annotation.Nullable;
import androidx.appcompat.app.AppCompatActivity;
import com.hasin.magnetad.AdError;
import com.hasin.magnetad.AdListener;
import com.hasin.magnetad.AdLoadResult;
import com.hasin.magnetad.InterstitialAd;
import com.hasin.magnetad.MagnetAdManager;

public class MainActivity extends AppCompatActivity {

    private InterstitialAd interstitialAd;
    private String cachedAdId;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        interstitialAd = MagnetAdManager.getInterstitialAd("YOUR-PLACEMENT-ID");

        interstitialAd.setAdListener(new AdListener() {
            @Override
            public void onAdShown() { /* the ad appeared on screen */ }

            @Override
            public void onAdDismissed() { /* resume the game/app */ }

            @Override
            public void onAdFailedToShow(AdError error) {
                Log.w("MagnetAd", "show failed: " + error.getCode());
                // Resume here too, exactly as in onAdDismissed
            }
        });

        interstitialAd.requestAd(this, result -> {
            if (result instanceof AdLoadResult.Success) {
                cachedAdId = ((AdLoadResult.Success) result).getAdId();
                interstitialAd.show(MainActivity.this, cachedAdId);
            } else {
                AdError error = ((AdLoadResult.Failure) result).getError();
                Log.w("MagnetAd", "load failed: " + error.getCode());
            }
        });
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (interstitialAd != null) {
            interstitialAd.destroy();
            interstitialAd = null;
        }
    }
}
```

And initialization in the `Application` class:

```java
MagnetAdManager.initialize(this, new AdConfig("YOUR-APP-ID", BuildConfig.DEBUG));

// Or with a handshake-completion callback. The parameter is a Kotlin function
// type, so the Java lambda must return Unit.INSTANCE:
MagnetAdManager.initialize(this, new AdConfig("YOUR-APP-ID", BuildConfig.DEBUG), success -> {
    Log.d("MagnetAd", "init success = " + success);
    return kotlin.Unit.INSTANCE;
});
```

> Using lambdas in Java requires `compileOptions` set to `JavaVersion.VERSION_11` (or at least 8) — the same as listed in [Requirements](#2-requirements).

---

## Step 6 — Events (AdListener) and the load result (AdLoadResult)

The load result no longer arrives through `AdListener`; the callback you pass to `requestAd` receives an `AdLoadResult`:

| `AdLoadResult` value | Meaning |
|---|---|
| `AdLoadResult.Success(adId)` | The ad and its creative (image or video) are ready; keep the `adId` and pass it to `show`. |
| `AdLoadResult.Failure(error)` | The load failed (network, server, no fill, …). |

`AdListener` reports only what happens **after `show`**:

| Event | When it fires |
|---|---|
| `onAdShown()` | The ad appeared on screen. |
| `onAdClicked()` | The user tapped the ad image (once per ad). **The ad closes immediately afterwards** and `onAdDismissed()` is called as well. |
| `onAdDismissed()` | The ad was closed — either via the ✕ button or following a user click. |
| `onAdFailedToShow(error)` | Showing was not possible (invalid/consumed `adId`, expiry, no available Activity, a render failure, or a video playback error). |
| `onRewarded()` | Rewarded placements only: the user watched the video far enough and the reward is due. |

> **Important for games:** if you resume your game or app from `onAdDismissed()`, make sure you do the same in `onAdFailedToShow(error)`. On a show failure the ad closes automatically but **only** `onAdFailedToShow` is called — `onAdDismissed` does not follow — and without this your app stays stuck in that case.

> All callbacks — both the `requestAd` callback and the `AdListener` events — are invoked on the **main thread**, so you can update the UI directly.

---

## Step 7 — Video and rewarded ads

### What changes in your code

**Nothing.** You call the same `getInterstitialAd`, `requestAd` and `show`. The only thing a rewarded placement needs from you is an `onRewarded()` override:

```kotlin
private lateinit var rewardedAd: InterstitialAd
private var rewardedAdId: String? = null

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)

    rewardedAd = MagnetAdManager.getInterstitialAd("YOUR-REWARDED-PLACEMENT-ID")

    rewardedAd.setAdListener(object : AdListener() {
        override fun onRewarded() {
            // The user watched the video; show the reward in your UI here
            showCoinAnimation()
        }

        override fun onAdDismissed() {
            rewardedAdId = null
            preloadRewarded()
        }

        override fun onAdFailedToShow(error: AdError) {
            rewardedAdId = null
        }
    })

    preloadRewarded()
}

private fun preloadRewarded() {
    rewardedAd.requestAd(this) { result ->
        rewardedAdId = (result as? AdLoadResult.Success)?.adId
    }
}

private fun onWatchForCoinsClicked() {
    rewardedAdId?.let { rewardedAd.show(this, it) }
}
```

### Telling the placement type apart in code

After a successful load you can read which creative type the server returned:

```kotlin
when (rewardedAd.placementType) {
    PlacementType.REWARDED     -> { /* rewarded ad */ }
    PlacementType.INTERSTITIAL -> { /* regular ad */ }
    null                       -> { /* nothing loaded yet */ }
}
```

---

## Controlling video volume

You set the starting volume in `AdConfig` and can change it at any time, including mid-playback:

```kotlin
MagnetAdManager.setVideoVolume(0.5f)   // 0f to 1f; values outside the range are clamped
MagnetAdManager.getVideoVolume()       // current value

MagnetAdManager.setVideoMuted(true)    // mute without losing the previous value
MagnetAdManager.isVideoMuted()         // current state
```

> `setVideoMuted(false)` restores the volume that was set before muting. These methods are callable from any thread.

> If your game has a sound toggle, pass its state through `AdConfig.videoVolume` at `initialize` and keep it in sync with these methods afterwards. Changing the `AdConfig` field directly after `initialize` has no effect.

---

## Using several placements at once

If you have both a regular and a rewarded placement, create a separate instance for each and keep both as fields. The instances are fully independent: each has its own cache, callbacks and lifecycle, and loading them at the same time causes no interference.

```kotlin
private lateinit var interstitialAd: InterstitialAd
private lateinit var rewardedAd: InterstitialAd

interstitialAd = MagnetAdManager.getInterstitialAd("PLACEMENT-INTERSTITIAL")
rewardedAd     = MagnetAdManager.getInterstitialAd("PLACEMENT-REWARDED")
```

> Show only **one** ad at a time. The SDK does not stop you from showing two at once, and the result is two stacked dialogs.

> Call `destroy()` on both instances in `onDestroy`.

---

## Full API reference

```kotlin
object MagnetAdManager {
    fun initialize(
        application: Application,
        config: AdConfig,
        onInitComplete: ((success: Boolean) -> Unit)? = null
    )
    fun getInterstitialAd(placementId: String): InterstitialAd
    fun isInitialized(): Boolean
    fun getSDKVersion(): String
    fun getConfig(): AdConfig?

    fun setVideoVolume(volume: Float)
    fun getVideoVolume(): Float
    fun setVideoMuted(muted: Boolean)
    fun isVideoMuted(): Boolean

    fun clearCache()
    fun shutdown()
    fun onTrimMemory(level: Int)
}

class InterstitialAd {
    val state: AdState                                        // IDLE / LOADING / LOADED / SHOWING / DESTROYED
    val placementType: PlacementType?                         // filled in after a successful load

    fun setAdListener(listener: AdListener)
    fun isAdReady(): Boolean                                  // no need to hold on to the adId yourself
    fun requestAd(
        context: Context,
        timeoutMs: Long = DEFAULT_REQUEST_TIMEOUT_MS,         // defaults to 30s; overall cap on the load
        callback: AdLoadCallback                              // fun interface; usable from Java as a lambda
    )                                                         // non-blocking; result arrives in the callback
    fun show(activity: Activity, adId: String)                // shows the ad matching this adId
    fun destroy()                                             // cancels the load and releases resources
}

abstract class AdListener {
    open fun onAdShown()
    open fun onAdDismissed()
    open fun onAdClicked()
    open fun onAdFailedToShow(error: AdError)
    open fun onRewarded()
}

sealed class AdLoadResult {
    data class Success(val adId: String) : AdLoadResult()
    data class Failure(val error: AdError) : AdLoadResult()
}

enum class PlacementType { INTERSTITIAL, REWARDED }

enum class AdState { IDLE, LOADING, LOADED, SHOWING, DESTROYED }

data class AdConfig(
    val appId: String,
    var debugMode: Boolean = false,
    val cacheSizeMb: Int = 50,
    var videoVolume: Float = 1f
)

data class AdError(val code: String, val message: String)
```

| Method | Description |
|---|---|
| `initialize(app, config, onInitComplete?)` | Call once in `Application`. Non-blocking. |
| `getInterstitialAd(placementId)` | Creates a **new** ad object for the given placement. Throws if called before `initialize`, or with a blank ID. Hold on to the instance instead of rebuilding it for every show. |
| `state` | A live read of what the instance is holding, for detecting "the ad I thought was showing is gone" without waiting on a callback. |
| `placementType` | The creative type returned by the most recent successful load; `null` before that. |
| `isAdReady()` | Whether a valid, unexpired ad is currently cached and ready to show — without you having to hold on to the `adId`. |
| `requestAd(context, timeoutMs?, callback)` | Explicitly loads an ad in the background, under an overall time cap (30 seconds by default); past that the `callback` fires with `AdErrorCode.TIMEOUT`. The result (`AdLoadResult`) arrives only through this `callback`. Callable from any thread. The SDK never loads automatically. If a valid ad is already cached, the same `adId` is returned with no new network request (single-slot cache). If a load is already in flight (for example a double-tapped button), this call's `callback` is queued alongside it and receives that same load's result — it is not ignored. |
| `show(activity, adId)` | Shows the ad matching `adId`. Requires an active **Activity**. The `adId` must come from the most recent `AdLoadResult.Success` and must not have been consumed yet. |
| `destroy()` | Call in `onDestroy` to release resources. |
| `setVideoVolume(v)` / `setVideoMuted(b)` | Control video ad volume, even mid-playback. |
| `shutdown()` | Fully shuts the SDK down (rarely needed). |

---

## Error codes

These are the values that appear in `AdError.code`:

| Code | Meaning |
|---|---|
| `SDK_NOT_INITIALIZED` | Called before `initialize`. |
| `NETWORK_ERROR` | The server was unreachable, or a connection error occurred. |
| `SERVER_ERROR` | Server-side error (unsuccessful HTTP status). |
| `INVALID_RESPONSE` | The server response could not be parsed. |
| `ASSET_LOAD_FAILED` | Downloading the ad image or video failed. |
| `UNEXPECTED_AD_TYPE` | The server returned a creative type this SDK version does not support. |
| `NO_FILL` | No suitable ad was available at that moment. This is not an error and happens in normal operation; skip it silently and call `requestAd` again later. |
| `TIMEOUT` | The whole load (network request + creative download) exceeded `timeoutMs`, 30 seconds by default. |
| `AD_NOT_READY` | `show` was called with an `adId` that does not match the loaded ad — for example you have not called `requestAd` yet, or that same `adId` was already shown once. |
| `AD_EXPIRED` | The loaded ad has expired; call `requestAd` again. |
| `ACTIVITY_UNAVAILABLE` | The Activity was finishing or being destroyed. |
| `SHOW_FAILED` | Showing the ad failed. |
| `VIDEO_PLAYBACK_ERROR` | The video player reported an error. |
| `VIDEO_SERVER_DIED` | The system media playback service died. |
| `VIDEO_TIMEOUT` | Video playback stalled and the watchdog closed it. |
| `UNKNOWN` | Unspecified error. |

> Branch on `error.code` rather than parsing the message text — these code values are stable.

---

## How the SDK behaves internally

- **Non-blocking:** the network request and image download run in coroutines on background threads; the `requestAd` callback and the `AdListener` events come back on the main thread.
- **Creative preloading:** `AdLoadResult.Success` is only emitted once **the creative (image or video) has also been downloaded**, so showing is instant and never flashes a black screen. A video is downloaded in full to local storage and played from that file; there is no streaming.
- **Automatic renderer choice:** the SDK decides between the image and video path from the media type the server reports. Your code is identical either way.
- **No automatic loading:** the SDK never loads an ad by itself — not during `initialize`, and not after a `show`. You are fully in control of when a load happens and when an ad is shown; and each `adId` can only be shown once.
- **Single-slot cache:** at most one loaded ad exists in the cache at any moment. If you call `requestAd` again while a valid ad (unshown and unexpired) is already cached, no new network request is made; the same `adId` comes back in the `callback`.
- **Close button on image ads:** while the ad is showing, the close button (✕) first displays a **5-second countdown** and then becomes active; during that time the hardware Back button is disabled too.
- **Close and skip rules on video ads:** whether the user may skip, after how many seconds, and whether Back is enabled are all asserted by the server per fill, not decided by the SDK. For a rewarded placement the server withholds those until the ad is watched far enough.
- **Video playback:** screen orientation is locked for the duration and restored afterwards, the screen is kept on, and playback pauses and resumes with the host Activity. If the player stalls without reporting completion or an error, a watchdog closes the dialog and reports `VIDEO_TIMEOUT` rather than trapping the user.
- **Video cache and completion reporting:** downloaded videos are cached up to 100 MB with the oldest evicted first, and any single video is capped at 100 MB. The "watched" event is retried with backoff and persisted on the device when the network is down, so it survives the app being closed and is flushed on the next `requestAd`.
- **Ad expiry:** a loaded ad has a limited validity period; if you `show` too late you get `AD_EXPIRED` and must load again.
- **Image render failure:** if the ad image fails to render after `show`, the dialog **closes itself** and `ASSET_LOAD_FAILED` is reported through `onAdFailedToShow`; the user never sees a black screen or an English error message.
- **HTTPS only:** the SDK's requests go over `https` exclusively; the package adds no cleartext setting or `networkSecurityConfig` to your app.
- **Lifecycle and screen rotation:** the SDK manages the ad dialog's state across configuration changes (such as rotation) automatically; nothing extra is required from you.
- **Memory management:** forwarding `onTrimMemory` from your `Application` lets the image cache be released under memory pressure.
- **Genuine API 21 support:** device signal collection has its own compatible path on Android 5 and 6; the SDK does not crash on older versions.

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `SDK_NOT_INITIALIZED`, or an exception from `getInterstitialAd` | You did not call `MagnetAdManager.initialize(...)` in `Application.onCreate`, or your Application class is not registered in the manifest. |
| Loading always fails with `NETWORK_ERROR` | Check internet/server access. The SDK communicates over `https`, so if you have a custom `networkSecurityConfig`, make sure it does not block the service domain or restrict it with pinning. |
| `AD_NOT_READY` on `show` | Either the `adId` is invalid/wrong, or you called `show` before receiving `AdLoadResult.Success`, or that same `adId` was already consumed. Call `requestAd` first, then call `show` with the received `adId` inside its callback. |
| `AD_EXPIRED` on `show` | Too much time passed between loading and showing; call `requestAd` again in the error callback. |
| Loading returns `NO_FILL` | Not an integration error; the server simply had no suitable ad at that moment. Do not show the user an error, and retry later. If you **always** get `NO_FILL`, verify the App Id and Placement Id are correct and the placement is enabled in the panel. |
| Loading returns `TIMEOUT` | The whole load (ad request + creative download) did not finish within `timeoutMs`, 30 seconds by default, usually because the user's network is slow. Treat it like `NO_FILL`: skip the ad this time and try again later. |
| Loading returns `ASSET_LOAD_FAILED` | Downloading or decoding the creative failed — either the network dropped mid-download, or the file exceeds its cap (**10 MB** for images, **100 MB** for video). If it repeats for a specific campaign, report the creative's size to the Magnet team. |
| `onRewarded` never fires | Either the placement is not a rewarded one (check the Placement Id), or the user did not watch the video far enough. Set `debugMode = true` and read the logs. |
| Video takes a long time to start | The video is downloaded in full before it is shown, so the delay is in `requestAd`, not in `show`. Load the ad earlier. |
| Loading returns `VIDEO_TIMEOUT` | The player stalled. If it repeats, report the device model and the video format to the Magnet team. |
| The ad appeared but closed immediately | Image rendering failed and the SDK deliberately closed the dialog; `onAdFailedToShow` is called with code `ASSET_LOAD_FAILED`. |
| The game/app does not resume after an ad | You tied resuming only to `onAdDismissed`; put the same logic in `onAdFailedToShow` as well (see [Step 6](#step-6--events-adlistener-and-the-load-result-adloadresult)). |
| Gradle error: `requires compileSdk 34 or later` | Your project's compileSdk is below 34; raise it to `34` or higher. |
| The `AD_ID` permission must not be in my app | Remove it with `tools:node="remove"` — see [About the `AD_ID` permission](#about-the-ad_id-permission). |
| The `com.hasin:magnetad` artifact cannot be found | MagnetAd is not published to any Maven repository; remove the `implementation("com.hasin:magnetad:…")` line and use the AAR file method — see [Step 1](#step-1--adding-the-package-aar-file). |
| `NoClassDefFoundError` or `ClassNotFoundException` at runtime (the build was fine) | One of the side dependencies from step 1-2 is missing. Because the AAR has no POM file, none of them are resolved automatically; complete the list. |
| `Duplicate class com.hasin.magnetad...` error | More than one MagnetAd AAR is left in `app/libs/`; delete the older versions. |
| Old behavior persists after upgrading the SDK | You replaced the AAR file but the filename in `build.gradle.kts` still points at the previous version. |
| Gradle cannot download the dependencies | `dl.google.com` is unreachable; put the `maven { url = uri("https://maven.myket.ir/") }` mirror before `google()`. |
| `module was compiled with an incompatible version of Kotlin` error | Your project's Kotlin version is older than 1.9; update it. |
| SDK classes are stripped after enabling Minify | `consumer-rules.pro` normally covers this; if you still hit it, inspect the release build with logs. |
| Release build fails with an R8 error on `okhttp3`/`okio`/`org.conscrypt` | The required `-dontwarn` rules are inside the AAR; make sure you picked up an up-to-date version of the package. If it still happens, add those same `-dontwarn` rules temporarily to your own `proguard-rules.pro` and let us know. |
| In release builds every request returns `INVALID_RESPONSE` (debug is fine) | The `kotlinx.serialization` rules are not being applied — a sign that the package's `consumer-rules.pro` did not reach your build; check your AAR/artifact version. |
| The installed store does not open and the browser is used instead | The `<queries>` element was dropped during the merge; inspect your build's merged manifest (it must contain the three store packages). |

For easier debugging, set `debugMode = true` in `AdConfig` so the SDK's logs are printed to **logcat** under the `MagnetAd` tag.

---

## Limitations

- **Interstitial** and **Rewarded** fullscreen ads only. Banner and native ads are not supported in this version.
- Supported video formats: **MP4** and **WebM**. Adaptive streaming (HLS, DASH) is not supported.
- Maximum size per video: **100 MB**.
- Only one ad can be shown at a time.
- **Android** only (API 21 and above).

---

*SDK version: `1.8.0` — matching the value from `MagnetAdManager.getSDKVersion()`.*

*Package downloads and latest release: [github.com/MagnetAds/magad-android-sdk](https://github.com/MagnetAds/magad-android-sdk)*
