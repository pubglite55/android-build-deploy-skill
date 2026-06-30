---
name: android-build-deploy
description: Automates the full Android build-deploy cycle: detect project type (Flutter/Gradle/Xposed), set correct JAVA_HOME/ANDROID_HOME, build debug APK, install via adb, force-stop target processes, and optionally launch the app. Use when the user says "build and install", "rebuild", "deploy to device", or after code changes that need device testing.
---

# Android Build & Deploy

Use this skill to execute the full build→install→restart cycle for Android projects on a connected device.

## Core Rules

- Always detect project type before setting environment variables.
- Always verify device connection with `adb devices` before install.
- Always force-stop target processes after install to ensure clean state.
- Never skip the build verification step — confirm BUILD SUCCESSFUL before installing.
- Preserve exact error output when builds fail — do not truncate.

## Project Type Detection

Detect project type by checking for these files in the project root:

| Type | Indicator Files | Build Command |
|------|----------------|---------------|
| **Flutter** | `pubspec.yaml`, `lib/main.dart` | `flutter build apk --debug` |
| **Gradle (Kotlin)** | `build.gradle.kts`, `settings.gradle.kts` | `./gradlew assembleDebug` |
| **Gradle (Groovy)** | `build.gradle`, `settings.gradle` | `./gradlew assembleDebug` |
| **Xposed Module** | `build.gradle.kts` + `xposed_init` or LSPosed metadata | `./gradlew assembleDebug` |

## Environment Setup

### JAVA_HOME Selection

- **Gradle 8.x projects**: Use JDK 21 — `/usr/local/Cellar/openjdk@21/21.0.11/libexec/openjdk.jdk/Contents/Home`
- **Gradle 9.x / AGP 9.x projects**: Use JDK 22 — `/Library/Java/JavaVirtualMachines/jdk-22.jdk/Contents/Home`
- **Flutter projects**: Use system default or JDK 21

Detect Gradle version from `gradle/wrapper/gradle-wrapper.properties`:
```
distributionUrl=https\://services.gradle.org/distributions/gradle-8.2-bin.zip
```
If version < 9.0 → JDK 21. If version ≥ 9.0 → JDK 22.

### ANDROID_HOME

Always: `/Users/xiuxiu391/Library/Android/sdk`

### Build Commands by Project

**PixelWatchSpoof** (Xposed module):
```bash
export JAVA_HOME=/usr/local/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home
export ANDROID_HOME=/Users/xiuxiu391/Library/Android/sdk
cd /Users/xiuxiu391/Desktop/watch/PixelWatchSpoof
./gradlew assembleDebug 2>&1 | tail -5
```

**MotoBuds** (AGP 8.2 + Gradle 8.2 wrapper):
```bash
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk-22.jdk/Contents/Home
export ANDROID_HOME=/Users/xiuxiu391/Library/Android/sdk
cd /Users/xiuxiu391/Desktop/motobuds for miui/MotoBuds
./gradlew assembleDebug 2>&1 | tail -5
```

**TA Companion** (Flutter):
```bash
/Users/xiuxiu391/flutter/bin/flutter build apk --debug 2>&1 | tail -5
```
Working dir: `/Users/xiuxiu391/Desktop/idea/ta_companion`

## Workflow

1. **Detect project type** — check for indicator files in current or specified directory
2. **Set environment** — JAVA_HOME based on Gradle version, ANDROID_HOME fixed
3. **Build** — run appropriate build command, capture output
4. **Verify** — check for "BUILD SUCCESSFUL" in output
5. **Locate APK** — standard paths:
   - Gradle: `app/build/outputs/apk/debug/app-debug.apk`
   - Flutter: `build/app/outputs/flutter-apk/app-debug.apk`
6. **Check device** — `adb devices` must show exactly one device
7. **Install** — `adb install -r -t <apk-path>`
8. **Force-stop** — stop target app and related processes:
   - Xposed modules: `adb shell "am force-stop com.mi.health && am force-stop com.android.bluetooth"`
   - Regular apps: `adb shell am force-stop <package.name>`
9. **Launch** (optional) — `adb shell am start -n <package>/<activity>` or `adb shell monkey -p <package> -c android.intent.category.LAUNCHER 1`

## Error Handling

- **BUILD FAILED**: Read the error output, fix the issue, rebuild. Common causes:
  - Wrong JDK version → check gradle-wrapper.properties
  - Missing `local.properties` → create with `sdk.dir=/Users/xiuxiu391/Library/Android/sdk`
  - Kotlin compilation errors → check `.kt:` lines in output
- **Install fails**: Check device connection, verify APK exists, try `adb install -r -t` with `-t` flag
- **App crashes on launch**: Check logcat immediately after launch

## Log Capture on Crash

After a failed launch, immediately run:
```bash
adb logcat -d 2>&1 | grep -E "FATAL|Exception|Error" | tail -20
```

## Required Output

Always output:
1. Project type detected
2. Build result (SUCCESS/FAILED + error summary if failed)
3. APK path
4. Install result
5. Force-stop confirmation
6. Launch result (if applicable)
