# AGENTS.md - VLC for Android

## Quick Reference

### Build Commands

```bash
# JVM-only build (app code only, fetches LibVLC from Maven)
./gradlew assembleDebug          # Debug build
./gradlew assembleRelease        # Release build
./gradlew assembleSignedRelease  # Release with local keystore

# Full native build (requires Linux + Android SDK/NDK)
./buildsystem/compile.sh --init              # Setup only (clone deps, gradlew)
./buildsystem/compile.sh -l -a arm64         # Build LibVLC only
./buildsystem/compile.sh -ml -a arm64        # Build Medialibrary only
./buildsystem/compile.sh -a arm64            # Build everything (LibVLC + ML + App)

# VLC 4 builds (add -vlc4 flag)
./buildsystem/compile.sh --init -vlc4
./gradlew assembleDebug -PforceVlc4=true

# Remote Access web component
./buildsystem/compile-remoteaccess.sh --init  # Clone + npm install

# LeakCanary (optional)
./gradlew assembleDebug -PleakCanaryEnabled=true
```

### Tests

```bash
# Unit tests
./gradlew testDebugUnitTest

# Single test class
./gradlew :application:vlc-android:testDebugUnitTest --tests "org.videolan.vlc.SomeTest"

# UI instrumentation tests (requires device/emulator)
./gradlew installDebugAndroidTest
adb shell am instrument -w -m -e clearPackageData true \
  -e package org.videolan.vlc \
  org.videolan.vlc.debug.test/org.videolan.vlc.MultidexTestRunner
```

## Architecture

### Module Structure

- `libvlcjni/` - LibVLC JNI bindings (cloned from videolan, not committed in repo)
- `medialibrary/` - Media indexing library (Kotlin + native JNI)
- `application/app` - Final APK assembly module (depends on all others)
- `application/vlc-android` - Main VLC UI and services (the core module)
- `application/television` - Android TV / Leanback UI
- `application/resources` - Shared resources and utilities
- `application/tools` - Common tools/extensions
- `application/mediadb` - Database layer (Room)
- `application/remote-access-server` - Remote access server
- `application/remote-access-client` - Remote access web client (npm + assets)
- `application/moviepedia` - Movie metadata feature
- `application/donations` - Donation features
- `application/live-plot-graph` - Live graph visualization

### Key Entry Points

- `application/app/.../VLCApplication.kt` - Application class
- `application/vlc-android/.../StartActivity.kt` - Main entry, routes to TV or Mobile UI
- `application/vlc-android/.../PlaybackService.kt` - Core playback service (~2100 lines)
- `application/vlc-android/.../MediaParsingService.kt` - Media indexing service

## VLC 3 vs VLC 4

Controlled by `forceVlc4` Gradle property (or `-vlc4` flag in compile.sh). When true:
- Source sets switch: `vlc4/src` replaces `vlc3/src` in both `application/vlc-android` and `medialibrary`
- NDK version: 21 (VLC3) / 28 (VLC4)
- `minSdk`: 17 (VLC3) / 23 (VLC4)
- Different LibVLC and Medialibrary versions from Maven
- LibVLC JNI branch: `libvlcjni-3.x` (VLC3) / `master` (VLC4)

## Build Types

| Type | LibVLC Source | Use Case |
|------|---------------|----------|
| `debug` | Maven (prebuilt) | Development |
| `dev` | Local project build | Full local build |
| `release` | Maven (prebuilt) | Distribution |
| `signedRelease` | Maven (prebuilt) | Signed release |
| `vlcBundle` | Maven (prebuilt) | App bundle (minSdk 30) |

**Important**: `dev` build type depends on `:libvlcjni:libvlc` and `:medialibrary` as project dependencies. All other types use Maven artifacts.

## Toolchain Versions

- Gradle: 9.3.1 (pinned with SHA256 check in compile.sh)
- Kotlin: 2.2.10
- KSP: 2.3.2
- Android Gradle Plugin: 9.1.1
- compileSdk / targetSdk: 36
- Java target: 1.8

## Environment (Native Builds)

Requires Linux, `ANDROID_SDK` and `ANDROID_NDK` env vars. `compile.sh` generates `local.properties` and sets up `gradle.properties` (signing config). **Do not hand-edit `gradle.properties` if using `compile.sh`** — it will be overwritten.

## Gotchas

1. **libvlcjni is cloned, not committed** — `compile.sh` clones into `libvlcjni/` at a pinned hash from `https://code.videolan.org/videolan/libvlcjni.git`. `libvlcjni/` does not exist in a fresh clone.

2. **remoteaccess is cloned, not committed** — `compile-remoteaccess.sh` clones into `application/remote-access-client/remoteaccess/` at a pinned hash from `https://code.videolan.org/videolan/remoteaccess`. The web UI must be built separately before the Android build.

3. **Source set switching** — VLC3/VLC4 code lives in `vlc3/src/` and `vlc4/src/` directories within both `application/vlc-android` and `medialibrary`. The `forceVlc4` property determines which is compiled. Do not assume `src/` is the only source root.

4. **NDK version mismatch** — VLC3 requires NDK 21 (last supporting API 17). VLC4 uses NDK 28. `compile.sh` handles this via `-vlc4`.

5. **ABI flavors** — App has `abi` flavor dimension with splits: `x86`, `x86_64`, `armeabi-v7a`, `arm64-v8a`. Release builds enable ABI splits automatically.

6. **Test runner** — Uses `MultidexTestRunner` with AndroidX Orchestrator. Package data is cleared between tests. Unit tests use Robolectric and run with `-noverify` JVM arg.

7. **Version code scheme** — Uses `T M NN RR AA` format where last two digits encode ABI variant. See `build.gradle` comments.

8. **ProGuard** — Disabled for release builds (`minifyEnabled false`). ProGuard files exist but aren't active.

9. **compile.sh overwrites gradle.properties** — It regenerates `gradle.properties` with Jetifier/AndroidX flags and keystore config. Any manual edits will be lost.

10. **LeakCanary** — Available via `-PleakCanaryEnabled=true` Gradle property. Only enabled for `debug` and `dev` build types.
