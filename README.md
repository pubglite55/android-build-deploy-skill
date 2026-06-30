# Android Build & Deploy Skill for MiMoCode

Automates the full Android build→install→restart cycle. Detects your project type, sets the correct environment variables, builds the debug APK, installs it on a connected device, and optionally launches the app.

## Features

- **Auto-detect project type**: Flutter, Gradle (Kotlin/Groovy), or Xposed module
- **Smart JAVA_HOME selection**: JDK 21 for Gradle 8.x, JDK 22 for Gradle 9.x+
- **Full deploy cycle**: Build → Verify → Install → Force-stop → Launch
- **Pre-configured paths**: PixelWatchSpoof, MotoBuds, TA Companion
- **Error handling**: Common failure scenarios with recovery steps

## Installation

### Option 1: Clone to your project

```bash
cd your-project
git clone https://github.com/pubglite55/android-build-deploy-skill.git .mimocode/skills/android-build-deploy
```

### Option 2: Copy SKILL.md manually

```bash
mkdir -p .mimocode/skills/android-build-deploy
curl -o .mimocode/skills/android-build-deploy/SKILL.md \
  https://raw.githubusercontent.com/pubglite55/android-build-deploy-skill/main/SKILL.md
```

### Option 3: Global installation

```bash
mkdir -p ~/.mimocode/skills/android-build-deploy
curl -o ~/.mimocode/skills/android-build-deploy/SKILL.md \
  https://raw.githubusercontent.com/pubglite55/android-build-deploy-skill/main/SKILL.md
```

## How It Works

### 1. Project Type Detection

The skill detects your project type by checking for indicator files:

| Project Type | Indicator Files | Build Command |
|-------------|-----------------|---------------|
| **Flutter** | `pubspec.yaml`, `lib/main.dart` | `flutter build apk --debug` |
| **Gradle (Kotlin)** | `build.gradle.kts`, `settings.gradle.kts` | `./gradlew assembleDebug` |
| **Gradle (Groovy)** | `build.gradle`, `settings.gradle` | `./gradlew assembleDebug` |
| **Xposed Module** | `build.gradle.kts` + `xposed_init` | `./gradlew assembleDebug` |

### 2. Environment Setup

**JAVA_HOME** is automatically selected based on Gradle version:

```
Gradle < 9.0  →  JDK 21
                 /usr/local/Cellar/openjdk@21/21.0.11/libexec/openjdk.jdk/Contents/Home

Gradle >= 9.0  →  JDK 22
                   /Library/Java/JavaVirtualMachines/jdk-22.jdk/Contents/Home
```

**ANDROID_HOME** is always:
```
/Users/xiuxiu391/Library/Android/sdk
```

### 3. Build & Deploy Cycle

```
┌─────────────────────────────────────────────────────────────┐
│  1. Detect Project Type                                     │
│     └─ Check for pubspec.yaml, build.gradle.kts, etc.      │
├─────────────────────────────────────────────────────────────┤
│  2. Set Environment                                         │
│     └─ JAVA_HOME (based on Gradle version)                  │
│     └─ ANDROID_HOME (fixed)                                 │
├─────────────────────────────────────────────────────────────┤
│  3. Build                                                   │
│     └─ Run gradlew assembleDebug or flutter build           │
│     └─ Verify BUILD SUCCESSFUL                              │
├─────────────────────────────────────────────────────────────┤
│  4. Locate APK                                              │
│     └─ Gradle: app/build/outputs/apk/debug/app-debug.apk   │
│     └─ Flutter: build/app/outputs/flutter-apk/app-debug.apk │
├─────────────────────────────────────────────────────────────┤
│  5. Install                                                 │
│     └─ adb install -r -t <apk-path>                        │
├─────────────────────────────────────────────────────────────┤
│  6. Force-stop                                              │
│     └─ Stop target app and related processes                │
├─────────────────────────────────────────────────────────────┤
│  7. Launch (optional)                                       │
│     └─ adb shell am start -n <package>/<activity>           │
└─────────────────────────────────────────────────────────────┘
```

## Pre-configured Projects

The skill includes optimized commands for these projects:

### PixelWatchSpoof (Xposed Module)

```bash
export JAVA_HOME=/usr/local/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home
export ANDROID_HOME=/Users/xiuxiu391/Library/Android/sdk
cd /Users/xiuxiu391/Desktop/watch/PixelWatchSpoof
./gradlew assembleDebug 2>&1 | tail -5
```

### MotoBuds (AGP 8.2 + Gradle 8.2)

```bash
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk-22.jdk/Contents/Home
export ANDROID_HOME=/Users/xiuxiu391/Library/Android/sdk
cd /Users/xiuxiu391/Desktop/motobuds for miui/MotoBuds
./gradlew assembleDebug 2>&1 | tail -5
```

### TA Companion (Flutter)

```bash
/Users/xiuxiu391/flutter/bin/flutter build apk --debug 2>&1 | tail -5
```

## Usage Triggers

The skill activates when you say:

- "Build and install"
- "Rebuild the app"
- "Deploy to device"
- "Build and deploy"
- After code changes that need device testing

## Error Handling

### BUILD FAILED

Common causes and fixes:

| Error | Cause | Fix |
|-------|-------|-----|
| `HasConvention` crash | Gradle 9.x with old AGP | Use JDK 22 or downgrade Gradle |
| `Unsupported class file major version` | JDK 22 with Gradle 8.x | Use JDK 21 |
| Missing `local.properties` | First build | Create with `sdk.dir=/Users/xiuxiu391/Library/Android/sdk` |
| Kotlin compilation errors | Code issues | Check `.kt:` lines in output |

### Install Fails

```bash
# Check device connection
adb devices

# Try with -t flag for test APKs
adb install -r -t <apk-path>

# Force install if version conflict
adb install -r -t -d <apk-path>
```

### App Crashes on Launch

Immediately capture crash logs:

```bash
adb logcat -d 2>&1 | grep -E "FATAL|Exception|Error" | tail -20
```

## Requirements

- Android SDK installed at `~/Library/Android/sdk`
- JDK 21 (for Gradle 8.x projects)
- JDK 22 (for Gradle 9.x projects)
- `adb` in PATH or full path to `platform-tools/adb`
- USB-connected Android device with USB debugging enabled

## Customization

### Add Your Project

Edit the SKILL.md and add your project's build command to the "Build Commands by Project" section:

```markdown
**MyProject**:
\`\`\`bash
export JAVA_HOME=/path/to/jdk
export ANDROID_HOME=/Users/xiuxiu391/Library/Android/sdk
cd /path/to/my/project
./gradlew assembleDebug 2>&1 | tail -5
\`\`\`
```

### Change JAVA_HOME Paths

Edit the "JAVA_HOME Selection" section in SKILL.md to match your system.

## License

MIT
