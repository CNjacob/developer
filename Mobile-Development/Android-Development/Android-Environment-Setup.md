# Android 环境搭建

## 1. 学习目标

这篇文档面向 Android 初学者，目标是把开发环境从零搭起来，并理解每个工具的作用。

完成后你应该能做到：

- 安装 Android Studio。
- 创建 Android 项目。
- 配置 Android SDK。
- 创建并运行模拟器。
- 连接真机调试。
- 使用 Gradle 构建项目。
- 使用 ADB 查看设备、安装应用、查看日志。
- 看懂常见环境报错。

## 2. Android 开发工具链总览

```text
Android Studio
    |
    |-- JDK: 编译 Java/Kotlin 代码
    |-- Android SDK: Android 平台 API、构建工具、模拟器镜像
    |-- Gradle: 构建系统
    |-- Android Gradle Plugin: Gradle 和 Android 构建流程的桥
    |-- Emulator: 模拟器
    |-- ADB: 连接设备、安装、调试、查看日志
```

| 工具 | 作用 |
|------|------|
| Android Studio | 官方 IDE，集成编辑器、SDK 管理、模拟器、调试器 |
| Android SDK | Android 开发工具包，包含平台 API、build-tools、platform-tools |
| JDK | Java Development Kit，用于编译 JVM 代码 |
| Gradle | 构建工具，负责依赖、编译、打包、测试 |
| Android Gradle Plugin | Android 官方 Gradle 插件，负责 APK/AAB 构建 |
| ADB | Android Debug Bridge，用于和手机/模拟器通信 |
| Emulator | 模拟 Android 设备，便于本地调试 |

## 3. 安装前准备

建议电脑配置：

| 项目 | 建议 |
|------|------|
| 内存 | 16GB 及以上更舒服，8GB 可以入门 |
| 磁盘 | 至少预留 20GB，SDK 和模拟器镜像会占空间 |
| 系统 | macOS、Windows、Linux 都可以 |
| CPU | 支持硬件虚拟化，模拟器性能更好 |

macOS 建议先安装：

```bash
# Xcode 命令行工具，提供 git 等基础工具
xcode-select --install

# 如果已经安装 Homebrew，可用它安装常用命令行工具
brew install git tree jq
```

Windows 建议：

- 安装 Git for Windows。
- 开启 BIOS/UEFI 中的虚拟化支持。
- 如果使用模拟器，按 Android Studio 提示安装相关 hypervisor 驱动。

## 4. 安装 Android Studio

Android Studio 是官方推荐 IDE，直接从官网下载安装。

安装后首次启动，按向导选择：

```text
Standard Setup
    |
    |-- Android SDK
    |-- Android SDK Platform
    |-- Android Emulator
    |-- Android SDK Platform-Tools
```

如果你不知道怎么选，初学阶段使用默认选项即可。

## 5. Android SDK 配置

打开 Android Studio：

```text
Settings / Preferences
    |
    v
Languages & Frameworks
    |
    v
Android SDK
```

需要关注三个页面：

| 页面 | 作用 |
|------|------|
| SDK Platforms | 安装不同 Android API 版本 |
| SDK Tools | 安装构建工具、平台工具、模拟器 |
| SDK Update Sites | SDK 下载源配置 |

初学建议安装：

| 类型 | 建议 |
|------|------|
| SDK Platform | 最新稳定 Android API，再加项目要求的最低 API |
| Android SDK Build-Tools | 默认随项目安装即可 |
| Android SDK Platform-Tools | 必装，提供 `adb` |
| Android Emulator | 需要模拟器时安装 |
| Android SDK Command-line Tools | 命令行构建和 CI 常用 |

## 6. 环境变量

Android Studio 通常会自动管理 SDK。命令行使用 `adb` 时，建议配置环境变量。

macOS / Linux 在 `~/.zshrc` 或 `~/.bashrc` 中加入：

```bash
# Android SDK 根目录，macOS 默认常见位置
export ANDROID_HOME="$HOME/Library/Android/sdk"
export ANDROID_SDK_ROOT="$ANDROID_HOME"

# platform-tools 中包含 adb
export PATH="$ANDROID_HOME/platform-tools:$PATH"

# emulator 命令
export PATH="$ANDROID_HOME/emulator:$PATH"

# cmdline-tools 中包含 sdkmanager、avdmanager
export PATH="$ANDROID_HOME/cmdline-tools/latest/bin:$PATH"
```

重新加载：

```bash
source ~/.zshrc
```

检查：

```bash
adb version
```

Windows 常见 SDK 路径：

```text
C:\Users\<你的用户名>\AppData\Local\Android\Sdk
```

把下面目录加入 Path：

```text
%LOCALAPPDATA%\Android\Sdk\platform-tools
%LOCALAPPDATA%\Android\Sdk\emulator
%LOCALAPPDATA%\Android\Sdk\cmdline-tools\latest\bin
```

## 7. 创建第一个项目

Android Studio 中选择：

```text
New Project
    |
    v
Empty Activity
```

推荐初学配置：

| 配置 | 建议 |
|------|------|
| Language | Kotlin |
| Minimum SDK | API 23 或按实际项目要求 |
| Build configuration language | Kotlin DSL |
| UI | Jetpack Compose 或传统 XML 均可 |

如果你是完全新手：

- 想学现代 Android：选 Kotlin + Jetpack Compose。
- 想看懂老项目：还要学习 Java + XML View。

## 8. 项目结构

一个典型 Android 项目：

```text
MyApp/
  settings.gradle.kts
  build.gradle.kts
  gradle/
  gradlew
  gradlew.bat
  app/
    build.gradle.kts
    src/
      main/
        AndroidManifest.xml
        java/ 或 kotlin/
        res/
          drawable/
          mipmap/
          values/
```

| 文件/目录 | 作用 |
|-----------|------|
| `settings.gradle.kts` | 声明项目和模块 |
| 根 `build.gradle.kts` | 项目级构建配置 |
| `app/build.gradle.kts` | App 模块构建配置 |
| `gradlew` | Gradle Wrapper，保证团队使用同一 Gradle 版本 |
| `AndroidManifest.xml` | App 清单，声明组件、权限、主题 |
| `src/main/java` 或 `src/main/kotlin` | 业务代码 |
| `res/` | 图片、字符串、颜色、布局等资源 |

## 9. Gradle 基础

Gradle 是 Android 的构建系统。日常命令建议用项目自带的 Gradle Wrapper：

```bash
# macOS / Linux
./gradlew tasks

# Windows
gradlew.bat tasks
```

常用命令：

```bash
# 构建 Debug APK
./gradlew assembleDebug

# 安装 Debug 包到连接的设备
./gradlew installDebug

# 执行单元测试
./gradlew test

# 执行 Android Lint
./gradlew lint

# 清理构建产物
./gradlew clean

# 查看依赖
./gradlew app:dependencies
```

常见 `app/build.gradle.kts` 示例：

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.example.myapp"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = 23
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"
    }
}

dependencies {
    implementation("androidx.core:core-ktx:1.13.1")
    implementation("androidx.appcompat:appcompat:1.7.0")
}
```

说明：

- `compileSdk`：用哪个 Android API 编译。
- `minSdk`：最低支持 Android 版本。
- `targetSdk`：声明已经适配到哪个 Android 行为版本。
- `applicationId`：应用唯一 ID，上架和安装识别它。
- `namespace`：代码和资源生成时使用的命名空间。

## 10. 模拟器配置

打开：

```text
Tools
    |
    v
Device Manager
    |
    v
Create Device
```

推荐初学选择：

| 项目 | 建议 |
|------|------|
| 设备 | Pixel 系列 |
| 系统镜像 | 最新稳定 API，优先 x86_64 或 arm64-v8a |
| 存储 | 默认即可 |
| 启动方式 | Cold Boot 可排查状态问题 |

模拟器常见问题：

| 问题 | 处理 |
|------|------|
| 启动慢 | 确认硬件虚拟化开启 |
| 黑屏 | Cold Boot 或重新创建模拟器 |
| 占空间大 | 删除不用的 AVD 和系统镜像 |
| 网络异常 | 重启模拟器或清理代理设置 |

命令行查看模拟器：

```bash
emulator -list-avds
```

启动指定模拟器：

```bash
emulator -avd Pixel_8_API_35
```

## 11. 真机调试

Android 手机操作：

```text
设置
  |
  v
关于手机
  |
  v
连续点击版本号 7 次
  |
  v
开发者选项
  |
  v
开启 USB 调试
```

连接电脑后检查：

```bash
adb devices
```

第一次连接手机会弹出授权窗口，点击允许。

如果设备显示 `unauthorized`：

```bash
adb kill-server
adb start-server
adb devices
```

然后重新插拔 USB 并确认授权。

## 12. ADB 常用命令

ADB 是 Android 调试必备工具。

```bash
# 查看设备
adb devices

# 安装 APK
adb install app-debug.apk

# 覆盖安装
adb install -r app-debug.apk

# 卸载应用
adb uninstall com.example.myapp

# 进入设备 shell
adb shell

# 查看日志
adb logcat

# 只看某个 tag
adb logcat -s MainActivity

# 清空日志
adb logcat -c

# 截图
adb shell screencap -p /sdcard/screen.png
adb pull /sdcard/screen.png .

# 录屏
adb shell screenrecord /sdcard/demo.mp4
adb pull /sdcard/demo.mp4 .

# 查看当前前台 Activity
adb shell dumpsys activity top
```

常见安装失败：

| 报错 | 原因 |
|------|------|
| `INSTALL_FAILED_VERSION_DOWNGRADE` | 安装版本号比已安装版本低 |
| `INSTALL_FAILED_UPDATE_INCOMPATIBLE` | 签名不一致 |
| `INSTALL_FAILED_INSUFFICIENT_STORAGE` | 设备空间不足 |

## 13. Logcat 日志

Kotlin 示例：

```kotlin
import android.util.Log

class UserRepository {
    fun loadUser() {
        // d 表示 debug 日志，适合开发阶段排查问题
        Log.d("UserRepository", "start load user")

        // e 表示 error 日志，适合记录异常
        try {
            error("network failed")
        } catch (e: Exception) {
            Log.e("UserRepository", "load user failed", e)
        }
    }
}
```

Java 示例：

```java
import android.util.Log;

public class UserRepository {
    public void loadUser() {
        // d 表示 debug 日志
        Log.d("UserRepository", "start load user");

        try {
            throw new RuntimeException("network failed");
        } catch (Exception e) {
            // 第三个参数传异常对象，Logcat 会显示堆栈
            Log.e("UserRepository", "load user failed", e);
        }
    }
}
```

命令行过滤：

```bash
adb logcat -s UserRepository
```

## 14. 常用 Android Studio 功能

| 功能 | 用途 |
|------|------|
| Project 视图 | 查看项目文件 |
| Logcat | 查看运行日志 |
| Debugger | 断点调试 |
| Layout Inspector | 查看界面层级 |
| Profiler | 分析 CPU、内存、网络、电量 |
| App Inspection | 查看数据库、后台任务等 |
| Device Manager | 管理模拟器 |
| SDK Manager | 管理 SDK |
| Build Analyzer | 分析构建耗时 |

常用快捷键：

| 操作 | macOS | Windows/Linux |
|------|-------|---------------|
| 搜索全部 | Shift Shift | Shift Shift |
| 格式化 | Option + Command + L | Ctrl + Alt + L |
| 查找类 | Command + O | Ctrl + N |
| 查找文件 | Shift + Command + O | Ctrl + Shift + N |
| 跳转定义 | Command + B | Ctrl + B |
| 运行 | Control + R | Shift + F10 |
| 调试 | Control + D | Shift + F9 |

## 15. 第一个 Kotlin Activity

```kotlin
package com.example.myapp

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.material3.Text

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // setContent 用 Compose 声明 UI
        setContent {
            Text(text = "Hello Android")
        }
    }
}
```

关键点：

- `Activity` 是一个界面入口。
- `onCreate` 是 Activity 创建时的生命周期回调。
- `setContent` 用于设置 Compose UI。
- `Text` 是 Compose 的文本组件。

## 16. 第一个 Java Activity

传统 XML + Java 示例。

`MainActivity.java`：

```java
package com.example.myapp;

import android.os.Bundle;
import android.widget.TextView;
import androidx.appcompat.app.AppCompatActivity;

public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // 创建一个 TextView，并设置显示文本
        TextView textView = new TextView(this);
        textView.setText("Hello Android");
        textView.setTextSize(24);

        // 把 TextView 作为当前页面内容
        setContentView(textView);
    }
}
```

## 17. 常见环境问题

### 17.1 adb: command not found

原因：`platform-tools` 没有加入 PATH。

检查 SDK 路径：

```bash
ls "$HOME/Library/Android/sdk/platform-tools"
```

加入 `~/.zshrc`：

```bash
export ANDROID_HOME="$HOME/Library/Android/sdk"
export PATH="$ANDROID_HOME/platform-tools:$PATH"
```

重新加载：

```bash
source ~/.zshrc
adb version
```

### 17.2 Gradle 下载很慢

建议：

- 保持网络稳定。
- 不要手动乱改 Gradle Wrapper。
- 团队项目优先使用仓库中的 `gradle/wrapper/gradle-wrapper.properties`。

查看 Gradle 版本：

```bash
./gradlew --version
```

### 17.3 SDK license 未接受

命令行接受 license：

```bash
sdkmanager --licenses
```

如果找不到 `sdkmanager`，确认安装了 Android SDK Command-line Tools。

### 17.4 模拟器无法启动

排查顺序：

```text
确认 CPU 虚拟化
        |
        v
更新 Android Emulator
        |
        v
Cold Boot
        |
        v
删除并重建 AVD
```

### 17.5 真机连接不上

排查：

```bash
adb kill-server
adb start-server
adb devices
```

同时确认：

- USB 调试已开启。
- 手机弹窗已授权。
- USB 线支持数据传输，不只是充电线。
- Windows 已安装对应 USB 驱动。

## 18. 最小验证清单

```bash
java -version
adb version
emulator -version
./gradlew --version
./gradlew assembleDebug
```

Android Studio 中确认：

- 能打开项目。
- SDK Manager 正常。
- Device Manager 能创建模拟器。
- Run 能安装到模拟器或真机。
- Logcat 能看到日志。

## 19. 学习路径

```text
环境搭建
  |
  v
Kotlin 或 Java 基础
  |
  v
Activity / 生命周期 / Intent
  |
  v
UI: Compose 或 XML View
  |
  v
网络 / 数据库 / 权限 / 存储
  |
  v
架构: ViewModel / Repository / UI State
  |
  v
调试 / 测试 / 性能 / 发布
```

## 20. 参考资料

- Android Studio 安装：https://developer.android.com/studio/install
- Android 入门：https://developer.android.com/get-started/overview
- Android SDK 工具：https://developer.android.com/tools
- Android Emulator：https://developer.android.com/studio/run/emulator
- ADB：https://developer.android.com/tools/adb

## 21. 总结

Android 环境搭建最重要的是理解工具边界：

| 问题 | 对应工具 |
|------|----------|
| 写代码 | Android Studio |
| 编译打包 | Gradle + Android Gradle Plugin |
| Android API | Android SDK Platform |
| 连接设备 | ADB |
| 本地调试设备 | Emulator / 真机 |
| 日志与断点 | Logcat / Debugger |

初学阶段不要急着安装很多插件。先保证能创建项目、运行到设备、查看日志、构建 APK，这就是最小闭环。
