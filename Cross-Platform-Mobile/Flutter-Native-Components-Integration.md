# iOS / Android 原生组件提供给 Flutter 使用

## 1. 适用场景

Flutter 可以覆盖大多数 UI 和业务开发，但遇到下面场景时，通常需要接入 iOS / Android 原生能力：

- Flutter 没有现成插件，或现有插件不满足业务需求。
- 需要调用系统 API，例如相机、蓝牙、定位、剪贴板、Keychain、Keystore。
- 需要复用已有 iOS / Android 原生 SDK。
- 需要展示原生 View，例如地图、广告、视频播放器、WebView、复杂自研控件。
- 需要接入平台生命周期、通知、后台任务、传感器。
- Flutter 模块嵌入已有原生 App，需要双向通信。

## 2. 核心方案总览

Flutter 和原生交互主要有三类方式：

| 方式 | 作用 | 典型场景 |
|------|------|----------|
| `MethodChannel` | Dart 主动调用原生方法，并接收一次性返回值 | 获取设备信息、调用支付、打开系统能力 |
| `EventChannel` | 原生持续推送事件给 Dart | 定位流、传感器、下载进度、播放器状态 |
| `BasicMessageChannel` | Dart 和原生互发消息 | 双向消息、轻量协议、非方法式通信 |
| `PlatformView` | 把原生 View 嵌入 Flutter 页面 | 地图、广告、视频、复杂原生 UI |
| Flutter Plugin | 把原生能力封装成可复用包 | 团队复用、发布到 pub、多个 App 共享 |
| Pigeon | 生成类型安全的 Dart / 原生通信代码 | 复杂接口、强类型、减少字符串错误 |

最常用组合：

```text
调用原生能力：MethodChannel
持续监听事件：EventChannel
展示原生控件：PlatformView
团队复用：Flutter Plugin
```

## 3. 架构关系

```text
Dart / Flutter
  |
  | MethodChannel / EventChannel / PlatformView
  v
Flutter Engine
  |
  +------------------+
  |                  |
  v                  v
Android Native      iOS Native
Kotlin / Java       Swift / Objective-C
```

注意：

- Dart 和原生代码运行在不同侧，通过 Flutter Engine 传递消息。
- Channel 名称必须两端一致。
- 参数和返回值必须是 Flutter 标准编解码器支持的类型。
- UI 更新必须在主线程。
- 原生能力涉及权限时，需要在原生侧和 Flutter 侧都设计好流程。

## 4. Channel 支持的数据类型

常见可直接传递的类型：

| Dart | Android | iOS |
|------|---------|-----|
| `null` | `null` | `nil` |
| `bool` | `Boolean` | `NSNumber` |
| `int` | `Integer` / `Long` | `NSNumber` |
| `double` | `Double` | `NSNumber` |
| `String` | `String` | `NSString` |
| `Uint8List` | `byte[]` | `FlutterStandardTypedData` |
| `List` | `ArrayList` | `NSArray` |
| `Map` | `HashMap` | `NSDictionary` |

建议：

- 简单数据用 `Map<String, dynamic>`。
- 复杂接口优先考虑 Pigeon。
- 不要传递超大对象；大文件传路径或分块。
- 错误用结构化错误码，不要只传字符串。

## 5. 方案一：直接在 Flutter App 宿主工程写原生代码

适合：

- 只服务当前 App。
- 原生能力比较少。
- 快速验证方案。

目录结构：

```text
flutter_app/
  lib/
    native/
      native_device_api.dart
  android/
    app/src/main/kotlin/com/example/app/MainActivity.kt
  ios/
    Runner/AppDelegate.swift
```

缺点：

- 不方便复用到其他 App。
- 原生代码散落在宿主工程。
- 后期能力变多时维护成本会上升。

## 6. MethodChannel：Flutter 调原生方法

示例目标：Flutter 调用原生方法获取设备信息。

### 6.1 Dart 侧封装

`lib/native/native_device_api.dart`：

```dart
import 'package:flutter/services.dart';

class NativeDeviceApi {
  // Channel 名称建议使用反向域名，避免和其他插件冲突
  static const MethodChannel _channel = MethodChannel(
    'com.example.native/device',
  );

  /// 获取原生设备信息。
  ///
  /// 返回示例：
  /// {
  ///   "platform": "android",
  ///   "model": "Pixel 8",
  ///   "systemVersion": "15"
  /// }
  static Future<Map<String, dynamic>> getDeviceInfo() async {
    try {
      final result = await _channel.invokeMapMethod<String, dynamic>(
        'getDeviceInfo',
      );

      // 原生侧可能返回 null，这里统一转为空 Map，避免 UI 层空判断过多
      return result ?? <String, dynamic>{};
    } on PlatformException catch (e) {
      // PlatformException 是原生侧 result.error(...) 映射过来的异常
      throw NativeCallException(
        code: e.code,
        message: e.message ?? 'Native call failed',
        details: e.details,
      );
    }
  }
}

class NativeCallException implements Exception {
  final String code;
  final String message;
  final Object? details;

  NativeCallException({
    required this.code,
    required this.message,
    this.details,
  });

  @override
  String toString() {
    return 'NativeCallException(code: $code, message: $message, details: $details)';
  }
}
```

页面中使用：

```dart
import 'package:flutter/material.dart';

class DeviceInfoPage extends StatefulWidget {
  const DeviceInfoPage({super.key});

  @override
  State<DeviceInfoPage> createState() => _DeviceInfoPageState();
}

class _DeviceInfoPageState extends State<DeviceInfoPage> {
  String _text = 'Not loaded';

  Future<void> _load() async {
    try {
      final info = await NativeDeviceApi.getDeviceInfo();

      setState(() {
        _text = info.toString();
      });
    } catch (e) {
      setState(() {
        _text = 'Error: $e';
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Native Device Info')),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            ElevatedButton(
              onPressed: _load,
              child: const Text('Load'),
            ),
            const SizedBox(height: 16),
            Text(_text),
          ],
        ),
      ),
    );
  }
}
```

### 6.2 Android Kotlin 侧实现

`android/app/src/main/kotlin/com/example/app/MainActivity.kt`：

```kotlin
package com.example.app

import android.os.Build
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.plugin.common.MethodChannel

class MainActivity : FlutterActivity() {
    companion object {
        // 必须和 Dart 侧 MethodChannel 名称完全一致
        private const val CHANNEL = "com.example.native/device"
    }

    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)

        MethodChannel(
            flutterEngine.dartExecutor.binaryMessenger,
            CHANNEL
        ).setMethodCallHandler { call, result ->
            when (call.method) {
                "getDeviceInfo" -> {
                    val info = mapOf(
                        "platform" to "android",
                        "model" to Build.MODEL,
                        "brand" to Build.BRAND,
                        "manufacturer" to Build.MANUFACTURER,
                        "systemVersion" to Build.VERSION.RELEASE,
                        "sdkInt" to Build.VERSION.SDK_INT
                    )

                    // success 会把 Map 编码后返回给 Dart
                    result.success(info)
                }

                else -> {
                    // Dart 调用了未实现的方法时返回 notImplemented
                    result.notImplemented()
                }
            }
        }
    }
}
```

### 6.3 iOS Swift 侧实现

`ios/Runner/AppDelegate.swift`：

```swift
import UIKit
import Flutter

@UIApplicationMain
@objc class AppDelegate: FlutterAppDelegate {
  private let channelName = "com.example.native/device"

  override func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
  ) -> Bool {
    let controller = window?.rootViewController as! FlutterViewController

    let channel = FlutterMethodChannel(
      name: channelName,
      binaryMessenger: controller.binaryMessenger
    )

    channel.setMethodCallHandler { [weak self] call, result in
      guard self != nil else {
        result(FlutterError(
          code: "APP_DEALLOCATED",
          message: "AppDelegate released",
          details: nil
        ))
        return
      }

      switch call.method {
      case "getDeviceInfo":
        let device = UIDevice.current

        let info: [String: Any] = [
          "platform": "ios",
          "model": device.model,
          "name": device.name,
          "systemName": device.systemName,
          "systemVersion": device.systemVersion
        ]

        // result 接收基础类型、数组、字典等标准可编码对象
        result(info)

      default:
        result(FlutterMethodNotImplemented)
      }
    }

    GeneratedPluginRegistrant.register(with: self)
    return super.application(application, didFinishLaunchingWithOptions: launchOptions)
  }
}
```

## 7. MethodChannel 参数传递

示例目标：Flutter 传入用户名，原生返回格式化结果。

### 7.1 Dart

```dart
static Future<String> formatUserName(String name) async {
  final result = await _channel.invokeMethod<String>(
    'formatUserName',
    <String, dynamic>{
      'name': name,
      'uppercase': true,
    },
  );

  return result ?? '';
}
```

### 7.2 Android Kotlin

```kotlin
"formatUserName" -> {
    // arguments 是 Dart 传来的 Map
    val args = call.arguments as? Map<*, *>
    val name = args?.get("name") as? String
    val uppercase = args?.get("uppercase") as? Boolean ?: false

    if (name.isNullOrBlank()) {
        result.error(
            "INVALID_ARGUMENT",
            "name is required",
            null
        )
        return@setMethodCallHandler
    }

    val formatted = if (uppercase) {
        name.uppercase()
    } else {
        name.trim()
    }

    result.success(formatted)
}
```

### 7.3 iOS Swift

```swift
case "formatUserName":
  guard
    let args = call.arguments as? [String: Any],
    let name = args["name"] as? String,
    !name.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty
  else {
    result(FlutterError(
      code: "INVALID_ARGUMENT",
      message: "name is required",
      details: nil
    ))
    return
  }

  let uppercase = args["uppercase"] as? Bool ?? false
  let formatted = uppercase
    ? name.uppercased()
    : name.trimmingCharacters(in: .whitespacesAndNewlines)

  result(formatted)
```

## 8. EventChannel：原生持续推送事件给 Flutter

示例目标：原生每秒推送一个计数值给 Flutter。

### 8.1 Dart

```dart
class NativeCounterStream {
  static const EventChannel _eventChannel = EventChannel(
    'com.example.native/counter',
  );

  static Stream<int> watchCounter() {
    return _eventChannel
        .receiveBroadcastStream()
        .map((event) => event as int);
  }
}
```

页面使用：

```dart
class CounterPage extends StatelessWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context) {
    return StreamBuilder<int>(
      stream: NativeCounterStream.watchCounter(),
      builder: (context, snapshot) {
        final value = snapshot.data;

        if (snapshot.hasError) {
          return Text('Error: ${snapshot.error}');
        }

        return Text('Counter: ${value ?? 0}');
      },
    );
  }
}
```

### 8.2 Android Kotlin

```kotlin
import android.os.Handler
import android.os.Looper
import io.flutter.plugin.common.EventChannel

class MainActivity : FlutterActivity() {
    private var counter = 0
    private var handler: Handler? = null
    private var runnable: Runnable? = null

    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)

        EventChannel(
            flutterEngine.dartExecutor.binaryMessenger,
            "com.example.native/counter"
        ).setStreamHandler(object : EventChannel.StreamHandler {
            override fun onListen(arguments: Any?, events: EventChannel.EventSink?) {
                handler = Handler(Looper.getMainLooper())

                runnable = object : Runnable {
                    override fun run() {
                        counter++

                        // success 会把事件推给 Dart Stream
                        events?.success(counter)

                        handler?.postDelayed(this, 1000)
                    }
                }

                handler?.post(runnable!!)
            }

            override fun onCancel(arguments: Any?) {
                // Flutter 侧取消监听时释放资源
                runnable?.let { handler?.removeCallbacks(it) }
                runnable = null
                handler = null
            }
        })
    }
}
```

### 8.3 iOS Swift

```swift
final class CounterStreamHandler: NSObject, FlutterStreamHandler {
  private var sink: FlutterEventSink?
  private var timer: Timer?
  private var counter = 0

  func onListen(
    withArguments arguments: Any?,
    eventSink events: @escaping FlutterEventSink
  ) -> FlutterError? {
    sink = events

    timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
      guard let self = self else { return }
      self.counter += 1
      events(self.counter)
    }

    return nil
  }

  func onCancel(withArguments arguments: Any?) -> FlutterError? {
    timer?.invalidate()
    timer = nil
    sink = nil
    return nil
  }
}
```

注册：

```swift
let counterChannel = FlutterEventChannel(
  name: "com.example.native/counter",
  binaryMessenger: controller.binaryMessenger
)

counterChannel.setStreamHandler(CounterStreamHandler())
```

## 9. PlatformView：把原生 View 嵌入 Flutter

适合：

- 地图。
- 广告。
- 原生播放器。
- 原生图表。
- 厂商 SDK 提供的 View。

不适合：

- 普通按钮、列表、文本这种 Flutter 能很好实现的 UI。
- 高频重绘、复杂叠加动画场景。

原因：PlatformView 会引入额外合成、手势、布局、生命周期成本。

## 10. Android 原生 View 提供给 Flutter

示例目标：Android 提供一个原生 `TextView`，Flutter 页面中嵌入显示。

### 10.1 Dart 侧

```dart
import 'dart:io';
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';

class NativeTextView extends StatelessWidget {
  final String text;

  const NativeTextView({
    super.key,
    required this.text,
  });

  @override
  Widget build(BuildContext context) {
    const viewType = 'com.example.native/text_view';

    final creationParams = <String, dynamic>{
      'text': text,
    };

    if (defaultTargetPlatform == TargetPlatform.android) {
      return AndroidView(
        viewType: viewType,
        creationParams: creationParams,
        creationParamsCodec: const StandardMessageCodec(),
      );
    }

    if (defaultTargetPlatform == TargetPlatform.iOS) {
      return UiKitView(
        viewType: viewType,
        creationParams: creationParams,
        creationParamsCodec: const StandardMessageCodec(),
      );
    }

    return const Text('Unsupported platform');
  }
}
```

使用：

```dart
SizedBox(
  height: 80,
  child: NativeTextView(text: 'Hello Native View'),
)
```

### 10.2 Android Native View

`NativeTextPlatformView.kt`：

```kotlin
package com.example.app

import android.content.Context
import android.graphics.Color
import android.view.View
import android.widget.TextView
import io.flutter.plugin.platform.PlatformView

class NativeTextPlatformView(
    context: Context,
    id: Int,
    creationParams: Map<String?, Any?>?
) : PlatformView {

    private val textView: TextView = TextView(context)

    init {
        val text = creationParams?.get("text") as? String ?: "Native Android View"

        textView.text = text
        textView.textSize = 20f
        textView.setTextColor(Color.WHITE)
        textView.setBackgroundColor(Color.rgb(33, 150, 243))
        textView.setPadding(32, 16, 32, 16)
    }

    override fun getView(): View {
        // Flutter 会把这个 Android View 嵌入到 Flutter 页面中
        return textView
    }

    override fun dispose() {
        // 如果 View 持有播放器、地图、传感器等资源，需要在这里释放
    }
}
```

### 10.3 Android ViewFactory

`NativeTextViewFactory.kt`：

```kotlin
package com.example.app

import android.content.Context
import io.flutter.plugin.common.StandardMessageCodec
import io.flutter.plugin.platform.PlatformView
import io.flutter.plugin.platform.PlatformViewFactory

class NativeTextViewFactory : PlatformViewFactory(StandardMessageCodec.INSTANCE) {
    override fun create(
        context: Context,
        viewId: Int,
        args: Any?
    ): PlatformView {
        val creationParams = args as? Map<String?, Any?>

        return NativeTextPlatformView(
            context = context,
            id = viewId,
            creationParams = creationParams
        )
    }
}
```

### 10.4 Android 注册 PlatformView

`MainActivity.kt`：

```kotlin
class MainActivity : FlutterActivity() {
    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)

        flutterEngine
            .platformViewsController
            .registry
            .registerViewFactory(
                "com.example.native/text_view",
                NativeTextViewFactory()
            )
    }
}
```

## 11. iOS 原生 View 提供给 Flutter

示例目标：iOS 提供一个原生 `UILabel`，Flutter 页面中嵌入显示。

### 11.1 iOS Native View

`NativeTextPlatformView.swift`：

```swift
import Flutter
import UIKit

final class NativeTextPlatformView: NSObject, FlutterPlatformView {
  private let label: UILabel

  init(
    frame: CGRect,
    viewIdentifier viewId: Int64,
    arguments args: Any?,
    binaryMessenger messenger: FlutterBinaryMessenger?
  ) {
    label = UILabel(frame: frame)
    super.init()

    let params = args as? [String: Any]
    let text = params?["text"] as? String ?? "Native iOS View"

    label.text = text
    label.textColor = .white
    label.font = .systemFont(ofSize: 20, weight: .medium)
    label.backgroundColor = UIColor.systemBlue
    label.textAlignment = .center
  }

  func view() -> UIView {
    // Flutter 会把这个 UIView 嵌入到 Flutter 页面中
    return label
  }
}
```

### 11.2 iOS ViewFactory

`NativeTextViewFactory.swift`：

```swift
import Flutter
import UIKit

final class NativeTextViewFactory: NSObject, FlutterPlatformViewFactory {
  private let messenger: FlutterBinaryMessenger

  init(messenger: FlutterBinaryMessenger) {
    self.messenger = messenger
    super.init()
  }

  func create(
    withFrame frame: CGRect,
    viewIdentifier viewId: Int64,
    arguments args: Any?
  ) -> FlutterPlatformView {
    return NativeTextPlatformView(
      frame: frame,
      viewIdentifier: viewId,
      arguments: args,
      binaryMessenger: messenger
    )
  }

  func createArgsCodec() -> FlutterMessageCodec & NSObjectProtocol {
    // 必须和 Dart 侧 creationParamsCodec 对应
    return FlutterStandardMessageCodec.sharedInstance()
  }
}
```

### 11.3 iOS 注册 PlatformView

`AppDelegate.swift`：

```swift
override func application(
  _ application: UIApplication,
  didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
) -> Bool {
  let controller = window?.rootViewController as! FlutterViewController

  let factory = NativeTextViewFactory(
    messenger: controller.binaryMessenger
  )

  registrar(forPlugin: "NativeTextViewPlugin")?
    .register(
      factory,
      withId: "com.example.native/text_view"
    )

  GeneratedPluginRegistrant.register(with: self)
  return super.application(application, didFinishLaunchingWithOptions: launchOptions)
}
```

## 12. 封装成 Flutter Plugin

当原生能力需要复用时，推荐封装成插件。

### 12.1 创建插件

```bash
flutter create \
  --template=plugin \
  --platforms=android,ios \
  --org com.example \
  native_tools
```

目录结构：

```text
native_tools/
  lib/
    native_tools.dart
    native_tools_platform_interface.dart
    native_tools_method_channel.dart
  android/
    src/main/kotlin/com/example/native_tools/NativeToolsPlugin.kt
  ios/
    Classes/
      NativeToolsPlugin.swift
  example/
    lib/main.dart
  pubspec.yaml
```

### 12.2 Dart 插件入口

`lib/native_tools.dart`：

```dart
import 'native_tools_platform_interface.dart';

class NativeTools {
  Future<String> getPlatformVersion() {
    return NativeToolsPlatform.instance.getPlatformVersion();
  }

  Future<Map<String, dynamic>> getBatteryInfo() {
    return NativeToolsPlatform.instance.getBatteryInfo();
  }
}
```

### 12.3 Platform Interface

`lib/native_tools_platform_interface.dart`：

```dart
import 'package:plugin_platform_interface/plugin_platform_interface.dart';

import 'native_tools_method_channel.dart';

abstract class NativeToolsPlatform extends PlatformInterface {
  NativeToolsPlatform() : super(token: _token);

  static final Object _token = Object();

  static NativeToolsPlatform _instance = MethodChannelNativeTools();

  static NativeToolsPlatform get instance => _instance;

  static set instance(NativeToolsPlatform instance) {
    PlatformInterface.verifyToken(instance, _token);
    _instance = instance;
  }

  Future<String> getPlatformVersion();

  Future<Map<String, dynamic>> getBatteryInfo();
}
```

### 12.4 MethodChannel 实现

`lib/native_tools_method_channel.dart`：

```dart
import 'package:flutter/foundation.dart';
import 'package:flutter/services.dart';

import 'native_tools_platform_interface.dart';

class MethodChannelNativeTools extends NativeToolsPlatform {
  @visibleForTesting
  final methodChannel = const MethodChannel('native_tools');

  @override
  Future<String> getPlatformVersion() async {
    final version = await methodChannel.invokeMethod<String>(
      'getPlatformVersion',
    );
    return version ?? '';
  }

  @override
  Future<Map<String, dynamic>> getBatteryInfo() async {
    final info = await methodChannel.invokeMapMethod<String, dynamic>(
      'getBatteryInfo',
    );
    return info ?? <String, dynamic>{};
  }
}
```

### 12.5 Android 插件实现

`android/src/main/kotlin/com/example/native_tools/NativeToolsPlugin.kt`：

```kotlin
package com.example.native_tools

import android.content.Context
import android.os.BatteryManager
import android.os.Build
import io.flutter.embedding.engine.plugins.FlutterPlugin
import io.flutter.plugin.common.MethodCall
import io.flutter.plugin.common.MethodChannel
import io.flutter.plugin.common.MethodChannel.MethodCallHandler
import io.flutter.plugin.common.MethodChannel.Result

class NativeToolsPlugin : FlutterPlugin, MethodCallHandler {
    private lateinit var channel: MethodChannel
    private lateinit var context: Context

    override fun onAttachedToEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        context = binding.applicationContext

        channel = MethodChannel(
            binding.binaryMessenger,
            "native_tools"
        )
        channel.setMethodCallHandler(this)
    }

    override fun onMethodCall(call: MethodCall, result: Result) {
        when (call.method) {
            "getPlatformVersion" -> {
                result.success("Android ${Build.VERSION.RELEASE}")
            }

            "getBatteryInfo" -> {
                val batteryManager = context.getSystemService(
                    Context.BATTERY_SERVICE
                ) as BatteryManager

                val level = batteryManager.getIntProperty(
                    BatteryManager.BATTERY_PROPERTY_CAPACITY
                )

                result.success(
                    mapOf(
                        "platform" to "android",
                        "level" to level
                    )
                )
            }

            else -> result.notImplemented()
        }
    }

    override fun onDetachedFromEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        channel.setMethodCallHandler(null)
    }
}
```

### 12.6 iOS 插件实现

`ios/Classes/NativeToolsPlugin.swift`：

```swift
import Flutter
import UIKit

public class NativeToolsPlugin: NSObject, FlutterPlugin {
  public static func register(with registrar: FlutterPluginRegistrar) {
    let channel = FlutterMethodChannel(
      name: "native_tools",
      binaryMessenger: registrar.messenger()
    )

    let instance = NativeToolsPlugin()
    registrar.addMethodCallDelegate(instance, channel: channel)
  }

  public func handle(
    _ call: FlutterMethodCall,
    result: @escaping FlutterResult
  ) {
    switch call.method {
    case "getPlatformVersion":
      result("iOS " + UIDevice.current.systemVersion)

    case "getBatteryInfo":
      UIDevice.current.isBatteryMonitoringEnabled = true

      let level = UIDevice.current.batteryLevel

      result([
        "platform": "ios",
        // batteryLevel 范围是 0.0 到 1.0，-1 表示不可用
        "level": level < 0 ? -1 : Int(level * 100)
      ])

    default:
      result(FlutterMethodNotImplemented)
    }
  }
}
```

### 12.7 在 App 中使用插件

`pubspec.yaml`：

```yaml
dependencies:
  native_tools:
    path: ../native_tools
```

Dart：

```dart
final nativeTools = NativeTools();

final version = await nativeTools.getPlatformVersion();
final battery = await nativeTools.getBatteryInfo();
```

## 13. 插件中注册 PlatformView

如果原生 View 要做成插件，需要在插件注册阶段注册 ViewFactory。

### 13.1 Android 插件注册 PlatformView

```kotlin
class NativeToolsPlugin : FlutterPlugin {
    override fun onAttachedToEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        binding
            .platformViewRegistry
            .registerViewFactory(
                "native_tools/text_view",
                NativeTextViewFactory()
            )
    }

    override fun onDetachedFromEngine(binding: FlutterPlugin.FlutterPluginBinding) {
        // 一般无需手动 unregister，单个 PlatformView 自己在 dispose 释放资源
    }
}
```

### 13.2 iOS 插件注册 PlatformView

```swift
public static func register(with registrar: FlutterPluginRegistrar) {
  let factory = NativeTextViewFactory(
    messenger: registrar.messenger()
  )

  registrar.register(
    factory,
    withId: "native_tools/text_view"
  )
}
```

## 14. 权限处理

原生能力经常涉及权限，例如相机、定位、蓝牙、相册。

建议流程：

```text
Flutter UI
  |
  v
检查权限状态
  |
  v
需要时请求权限
  |
  v
权限通过后调用原生能力
  |
  v
失败时返回结构化错误
```

### 14.1 Android 权限声明

`android/app/src/main/AndroidManifest.xml`：

```xml
<uses-permission android:name="android.permission.CAMERA" />
```

Android 运行时权限可以：

- 在 Flutter 侧用 `permission_handler` 统一处理。
- 在原生插件侧处理，并通过 Channel 返回结果。

如果插件自己处理权限，需要实现 Activity 相关接口，不能只依赖 `applicationContext`。

### 14.2 iOS 权限声明

`ios/Runner/Info.plist`：

```xml
<key>NSCameraUsageDescription</key>
<string>需要使用相机扫描二维码</string>
```

iOS 没有正确配置 usage description 时，访问受保护能力可能直接崩溃。

## 15. 线程与生命周期

### 15.1 线程原则

| 场景 | 建议 |
|------|------|
| 原生 UI 操作 | 主线程 |
| 耗时计算 | 后台线程 |
| 网络 / 文件 | 后台线程 |
| Channel 回调结果 | 确保生命周期有效，避免重复回调 |

Android：

```kotlin
CoroutineScope(Dispatchers.IO).launch {
    val data = loadLargeData()

    withContext(Dispatchers.Main) {
        result.success(data)
    }
}
```

iOS：

```swift
DispatchQueue.global(qos: .userInitiated).async {
  let data = self.loadLargeData()

  DispatchQueue.main.async {
    result(data)
  }
}
```

### 15.2 生命周期原则

- `EventChannel` 在 `onCancel` 中停止监听。
- `PlatformView.dispose()` 释放原生资源。
- Android 插件在 `onDetachedFromEngine` 清空 Channel handler。
- iOS 定时器、通知、代理要在不用时释放。
- 不要让原生长生命周期对象强持有 Activity / UIViewController。

## 16. 错误处理规范

建议统一错误结构：

| 字段 | 说明 |
|------|------|
| code | 机器可读错误码 |
| message | 人类可读说明 |
| details | 调试细节，可选 |

Android：

```kotlin
result.error(
    "PERMISSION_DENIED",
    "Camera permission denied",
    mapOf("permission" to "CAMERA")
)
```

iOS：

```swift
result(FlutterError(
  code: "PERMISSION_DENIED",
  message: "Camera permission denied",
  details: ["permission": "CAMERA"]
))
```

Dart：

```dart
try {
  await NativeDeviceApi.getDeviceInfo();
} on PlatformException catch (e) {
  debugPrint('code=${e.code}, message=${e.message}, details=${e.details}');
}
```

## 17. Pigeon：类型安全通信

当接口越来越多时，手写 MethodChannel 容易出现：

- 方法名拼错。
- 参数 key 拼错。
- 返回类型不一致。
- iOS / Android / Dart 三端协议不同步。

Pigeon 可以根据 Dart 接口定义生成 Dart、Android、iOS 通信代码。

### 17.1 添加依赖

`pubspec.yaml`：

```yaml
dev_dependencies:
  pigeon: ^latest
```

实际项目中应固定明确版本。

### 17.2 定义协议

`pigeons/native_api.dart`：

```dart
import 'package:pigeon/pigeon.dart';

class DeviceInfo {
  String platform;
  String model;
  String systemVersion;

  DeviceInfo({
    required this.platform,
    required this.model,
    required this.systemVersion,
  });
}

@HostApi()
abstract class NativeDeviceHostApi {
  DeviceInfo getDeviceInfo();
}
```

### 17.3 生成代码

```bash
dart run pigeon \
  --input pigeons/native_api.dart \
  --dart_out lib/src/native_api.g.dart \
  --kotlin_out android/src/main/kotlin/com/example/native_tools/NativeApi.g.kt \
  --kotlin_package com.example.native_tools \
  --swift_out ios/Classes/NativeApi.g.swift
```

生成后：

- Dart 调用生成的 API。
- Android 实现生成的接口。
- iOS 实现生成的协议。

适合：接口多、团队协作、需要强约束的插件。

## 18. 调试与排查

### 18.1 Flutter 侧

```bash
flutter run -v
flutter logs
flutter doctor -v
```

Dart 中打印：

```dart
debugPrint('call native start');
```

### 18.2 Android 侧

```bash
adb logcat -s NativeToolsPlugin
```

Kotlin：

```kotlin
Log.d("NativeToolsPlugin", "method=${call.method}")
```

### 18.3 iOS 侧

在 Xcode 控制台查看日志：

```swift
print("NativeToolsPlugin method=\(call.method)")
```

常见问题：

| 问题 | 原因 |
|------|------|
| MissingPluginException | 插件未注册、Channel 名称不一致、热重启不够需全量重跑 |
| PlatformException | 原生主动返回错误 |
| 返回 null | 原生没调用 result 或类型不匹配 |
| iOS 崩溃 | Info.plist 权限缺失、强制类型转换失败 |
| Android 编译失败 | 包名、Gradle 配置、minSdk、依赖版本不匹配 |
| PlatformView 空白 | viewType 不一致、Factory 未注册、尺寸为 0 |

## 19. 性能注意事项

### 19.1 Channel 性能

- Channel 适合控制命令和小数据。
- 不适合高频、大量、逐帧数据传输。
- 图片、视频、大文件优先传文件路径、URI、纹理 ID。
- 频繁事件要节流或批量发送。

### 19.2 PlatformView 性能

- PlatformView 比 Flutter Widget 更重。
- 避免在长列表里大量嵌入原生 View。
- 注意手势冲突和滚动嵌套。
- 原生播放器、地图、广告要明确生命周期释放。

## 20. 推荐工程规范

### 20.1 Channel 命名

```text
com.company.project/module
```

示例：

```text
com.example.native/device
com.example.native/counter
com.example.native/text_view
```

### 20.2 Dart 封装

不要在页面里直接散落 `MethodChannel` 调用。

推荐：

```text
lib/
  native/
    native_device_api.dart
    native_error.dart
  features/
    device/
      device_info_page.dart
```

### 20.3 插件封装

推荐：

```text
native_tools/
  lib/
    native_tools.dart
    src/
      native_tools_platform_interface.dart
      native_tools_method_channel.dart
  android/
  ios/
  example/
```

### 20.4 API 设计

建议：

- 方法名用动词，例如 `getDeviceInfo`、`startScan`、`stopScan`。
- 参数用 Map 时，key 定义为常量或使用 Pigeon。
- 返回值结构稳定，不随意改变字段含义。
- 错误码文档化。
- 持续事件用 EventChannel，不要用 MethodChannel 轮询。

## 21. 完整操作步骤总结

### 21.1 App 内直接接入原生能力

```text
1. 在 Dart 定义 MethodChannel / EventChannel
2. 在 Android MainActivity configureFlutterEngine 注册 Channel
3. 在 iOS AppDelegate 注册 Channel
4. Dart 调用 invokeMethod 或监听 Stream
5. 原生处理参数并返回 success / error / notImplemented
6. Flutter 页面处理成功、失败、loading 状态
7. 真机验证权限、生命周期、异常路径
```

### 21.2 封装成插件

```text
1. flutter create --template=plugin --platforms=android,ios
2. 在 lib/ 设计 Dart API
3. 在 android/ 实现 FlutterPlugin 和 MethodCallHandler
4. 在 ios/ 实现 FlutterPlugin
5. 在 example/ 写最小可运行示例
6. 编写 README 和错误码说明
7. flutter test
8. 在 Android 和 iOS 真机分别验证
9. 需要强类型时引入 Pigeon
10. 多 App 复用时通过 path、git 或 pub 发布
```

### 21.3 提供原生 View

```text
1. Dart 侧用 AndroidView / UiKitView，定义 viewType
2. Android 实现 PlatformView
3. Android 实现 PlatformViewFactory
4. Android 注册 viewType
5. iOS 实现 FlutterPlatformView
6. iOS 实现 FlutterPlatformViewFactory
7. iOS 注册 viewType
8. Dart 传 creationParams
9. 原生 View 读取参数并渲染
10. dispose 中释放地图、播放器、广告等资源
```

## 22. 参考资料

- Flutter Platform Channels：https://docs.flutter.dev/platform-integration/platform-channels
- Flutter Android Platform Views：https://docs.flutter.dev/platform-integration/android/platform-views
- Flutter iOS Platform Views：https://docs.flutter.dev/platform-integration/ios/platform-views
- Flutter Developing Packages & Plugins：https://docs.flutter.dev/packages-and-plugins/developing-packages
- Flutter Pigeon：https://pub.dev/packages/pigeon

## 23. 总结

选择方式时可以按这个规则：

| 需求 | 推荐方式 |
|------|----------|
| Flutter 调一次原生能力 | MethodChannel |
| 原生持续推送状态 | EventChannel |
| 双向轻量消息 | BasicMessageChannel |
| 嵌入原生 UI | PlatformView |
| 多项目复用 | Flutter Plugin |
| 接口复杂、多人协作 | Pigeon |

不要为了“原生”而原生。Flutter 能稳定实现的 UI 和业务优先用 Flutter；只有系统能力、厂商 SDK、原生 View 或性能边界明确时，再通过 Channel / Plugin / PlatformView 接入。
