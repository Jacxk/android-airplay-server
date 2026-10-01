# AGENTS.md — android-airplay-server

Last updated: 2026-09-29

Fork of [jqssun/android-airplay-server](https://github.com/jqssun/android-airplay-server) (UxPlay-based AirPlay 2 receiver for Android / Android TV). Local remote: `origin` → `Jacxk/android-airplay-server`. Upstream: `upstream` (fetch-only).

## What this app is

Turns an Android device into an AirPlay display/speaker. Native RAOP/mDNS lives in C (UxPlay submodule); Kotlin/Compose is the UI + MediaCodec/ExoPlayer bridge.

```
Apple sender → UxPlay (C/JNI) → AirPlayService → UI / MediaCodec / AudioTrack / ExoPlayer
```

## Layout (where to edit)

| Path | Role |
|------|------|
| `app/src/main/kotlin/.../ui/` | Compose UI (`MainScreen.kt` = now-playing, tabs, fullscreen) |
| `app/src/main/kotlin/.../service/AirPlayService.kt` | Foreground service, session state, DACP hooks |
| `app/src/main/kotlin/.../audio/DacpController.kt` | Sender remote control (next/prev/pause) via DACP + mDNS |
| `app/src/main/kotlin/.../viewmodel/MainViewModel.kt` | UI ↔ service bridge |
| `app/src/main/cpp/` | Native bridge + CMake |
| `app/src/main/cpp/third_party/UxPlay` | **Git submodule** — do not commit dirty tree noise |
| `app/src/main/cpp/patches/UxPlay/` | Patches applied at configure time (`applyUxplayPatches`) |

## Target device (local)

Primary test device: Verizon Stream TV Soundbar (`Askey sti6251d315`), `armeabi-v7a` only.

```bash
# ABI filter is intentional for fast soundbar installs — restore full ABI list for releases
# app/build.gradle.kts → allAbis = listOf("armeabi-v7a")

adb devices -l
# often: 192.168.68.110:5555  product:sti6251d315

./gradlew :app:assembleDebug
adb install -r -d app/build/outputs/apk/debug/app-debug.apk
adb shell am start -n io.github.jqssun.airplay/.MainActivity
```

Use `adb install -r -d` when the device has a higher `versionCode` than the local APK.

Package: `io.github.jqssun.airplay`.

## Build notes

```bash
git submodule update --init --recursive   # first clone
./gradlew :app:assembleDebug
```

- Submodules: UxPlay, libplist, openssl-cmake, ffmpeg (see `.gitmodules`).
- If dex fails with `Type … is defined multiple times` and paths like `… 2.class`, macOS Finder duplicates polluted `app/build`. Fix: `./gradlew clean :app:assembleDebug` (or delete `app/build`). Same pattern under `UxPlay/lib/* 2.h` — leave those **uncommitted**.
- `applyUxplayPatches` runs before CMake configure; UxPlay showing `dirty` after a build is expected if patches/duplicates are present. Prefer not committing the submodule pointer unless intentionally updating UxPlay.

## Conventions for agents

- Prefer small, focused diffs. Match existing Kotlin/Compose style (no drive-by refactors).
- Do **not** commit `UxPlay` dirty submodule state, numbered Finder duplicates (`* 2.*`), or secrets (`local.properties`, keystores).
- Commit only when the user asks. Push only when asked. Prefer Conventional Commits (`fix(ui): …`, `fix(audio): …`).
- Now-playing UI lives in `FullscreenNowPlaying` inside `MainScreen.kt`. Zone contrast samples the **cropped** backdrop (`ContentScale.Crop`) per top/middle/bottom band; prefer white chrome unless contrast is too low.
- Transport buttons go through `DacpController`. After Home → resume, resolution can fail; `MainActivity.onResume` → `refreshAudioRemote()` / `ensureResolved()`. Raw mDNS queries use the QU (unicast-response) bit because the socket is not on port 5353.
- TV/D-pad: use `dpadFocus`, `requestFocusUntilLanded` (`Tv.kt` / `TvFocus.kt`).

## Useful logs

```bash
adb logcat -v time | rg -i 'DacpController|AirPlayService|DACP:'
```

## Upstream

- Do not force-push `main`. Feature work stays on branches (e.g. `feat/apple-music-nowplaying-tv`).
- Prefer `origin` for this fork; treat `upstream` as read-only.
