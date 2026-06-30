# Android 构建部署技能 - MiMoCode

自动化 Android 完整的构建→安装→重启流程。自动检测项目类型，设置正确的环境变量，构建 debug APK，通过 adb 安装到设备，并可选启动应用。

## 功能特性

- **自动检测项目类型**：Flutter、Gradle（Kotlin/Groovy）、Xposed 模块
- **智能 JAVA_HOME 选择**：Gradle 8.x 使用 JDK 21，Gradle 9.x+ 使用 JDK 22
- **完整部署周期**：构建 → 验证 → 安装 → 强制停止 → 启动
- **预配置路径**：PixelWatchSpoof、MotoBuds、TA Companion
- **错误处理**：常见失败场景及恢复步骤

## 安装方法

### 方式一：克隆到项目目录

```bash
cd 你的项目
git clone https://github.com/pubglite55/android-build-deploy-skill.git .mimocode/skills/android-build-deploy
```

### 方式二：手动复制 SKILL.md

```bash
mkdir -p .mimocode/skills/android-build-deploy
curl -o .mimocode/skills/android-build-deploy/SKILL.md \
  https://raw.githubusercontent.com/pubglite55/android-build-deploy-skill/main/SKILL.md
```

### 方式三：全局安装

```bash
mkdir -p ~/.mimocode/skills/android-build-deploy
curl -o ~/.mimocode/skills/android-build-deploy/SKILL.md \
  https://raw.githubusercontent.com/pubglite55/android-build-deploy-skill/main/SKILL.md
```

## 工作原理

### 1. 项目类型检测

技能通过检查特征文件来检测项目类型：

| 项目类型 | 特征文件 | 构建命令 |
|---------|---------|---------|
| **Flutter** | `pubspec.yaml`、`lib/main.dart` | `flutter build apk --debug` |
| **Gradle (Kotlin)** | `build.gradle.kts`、`settings.gradle.kts` | `./gradlew assembleDebug` |
| **Gradle (Groovy)** | `build.gradle`、`settings.gradle` | `./gradlew assembleDebug` |
| **Xposed 模块** | `build.gradle.kts` + `xposed_init` | `./gradlew assembleDebug` |

### 2. 环境配置

**JAVA_HOME** 根据 Gradle 版本自动选择：

```
Gradle < 9.0  →  JDK 21
                 /usr/local/Cellar/openjdk@21/21.0.11/libexec/openjdk.jdk/Contents/Home

Gradle >= 9.0  →  JDK 22
                   /Library/Java/JavaVirtualMachines/jdk-22.jdk/Contents/Home
```

**ANDROID_HOME** 固定为：
```
/Users/xiuxiu391/Library/Android/sdk
```

### 3. 构建部署流程

```
┌─────────────────────────────────────────────────────────────┐
│  1. 检测项目类型                                             │
│     └─ 检查 pubspec.yaml、build.gradle.kts 等特征文件        │
├─────────────────────────────────────────────────────────────┤
│  2. 设置环境变量                                             │
│     └─ JAVA_HOME（根据 Gradle 版本）                         │
│     └─ ANDROID_HOME（固定）                                  │
├─────────────────────────────────────────────────────────────┤
│  3. 构建                                                     │
│     └─ 运行 gradlew assembleDebug 或 flutter build          │
│     └─ 验证 BUILD SUCCESSFUL                                │
├─────────────────────────────────────────────────────────────┤
│  4. 定位 APK                                                 │
│     └─ Gradle: app/build/outputs/apk/debug/app-debug.apk   │
│     └─ Flutter: build/app/outputs/flutter-apk/app-debug.apk │
├─────────────────────────────────────────────────────────────┤
│  5. 安装                                                     │
│     └─ adb install -r -t <apk路径>                          │
├─────────────────────────────────────────────────────────────┤
│  6. 强制停止                                                 │
│     └─ 停止目标应用及相关进程                                 │
├─────────────────────────────────────────────────────────────┤
│  7. 启动（可选）                                             │
│     └─ adb shell am start -n <包名>/<Activity>              │
└─────────────────────────────────────────────────────────────┘
```

## 预配置项目

技能包含以下项目的优化命令：

### PixelWatchSpoof（Xposed 模块）

```bash
export JAVA_HOME=/usr/local/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home
export ANDROID_HOME=/Users/xiuxiu391/Library/Android/sdk
cd /Users/xiuxiu391/Desktop/watch/PixelWatchSpoof
./gradlew assembleDebug 2>&1 | tail -5
```

### MotoBuds（AGP 8.2 + Gradle 8.2）

```bash
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk-22.jdk/Contents/Home
export ANDROID_HOME=/Users/xiuxiu391/Library/Android/sdk
cd /Users/xiuxiu391/Desktop/motobuds for miui/MotoBuds
./gradlew assembleDebug 2>&1 | tail -5
```

### TA Companion（Flutter）

```bash
/Users/xiuxiu391/flutter/bin/flutter build apk --debug 2>&1 | tail -5
```

## 触发条件

以下场景会触发该技能：

- "构建并安装"
- "重新构建应用"
- "部署到设备"
- "构建部署"
- 代码修改后需要在设备上测试

## 错误处理

### 构建失败

常见原因及解决方案：

| 错误 | 原因 | 解决方案 |
|-----|------|---------|
| `HasConvention` 崩溃 | Gradle 9.x + 旧版 AGP | 使用 JDK 22 或降级 Gradle |
| `Unsupported class file major version` | JDK 22 + Gradle 8.x | 使用 JDK 21 |
| 缺少 `local.properties` | 首次构建 | 创建文件，内容：`sdk.dir=/Users/xiuxiu391/Library/Android/sdk` |
| Kotlin 编译错误 | 代码问题 | 检查输出中的 `.kt:` 行 |

### 安装失败

```bash
# 检查设备连接
adb devices

# 使用 -t 标志安装测试 APK
adb install -r -t <apk路径>

# 版本冲突时强制安装
adb install -r -t -d <apk路径>
```

### 启动崩溃

立即捕获崩溃日志：

```bash
adb logcat -d 2>&1 | grep -E "FATAL|Exception|Error" | tail -20
```

## 环境要求

- Android SDK 安装在 `~/Library/Android/sdk`
- JDK 21（Gradle 8.x 项目）
- JDK 22（Gradle 9.x 项目）
- `adb` 在 PATH 中，或使用完整路径 `platform-tools/adb`
- USB 连接的 Android 设备，已开启 USB 调试

## 自定义配置

### 添加你的项目

编辑 SKILL.md，在"构建命令"部分添加你的项目：

```markdown
**我的项目**：
\`\`\`bash
export JAVA_HOME=/path/to/jdk
export ANDROID_HOME=/Users/xiuxiu391/Library/Android/sdk
cd /path/to/my/project
./gradlew assembleDebug 2>&1 | tail -5
\`\`\`
```

### 修改 JAVA_HOME 路径

编辑 SKILL.md 中的"JAVA_HOME 选择"部分，修改为你系统的路径。

## 许可证

MIT
