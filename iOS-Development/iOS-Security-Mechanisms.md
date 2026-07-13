# iOS 安全机制总结

## 1. iOS 安全体系概述

iOS 的安全不是某一个 API 或某一种加固手段，而是一套从硬件、系统、应用沙盒、权限、网络、数据存储到运行时防护的分层体系。开发者通常能直接接触到的重点包括：

- **Keychain**：保存 Token、密码、证书、私钥等敏感信息
- **Data Protection**：基于设备锁屏状态保护本地文件
- **App Sandbox**：隔离不同 App 的进程、文件和系统资源访问
- **权限与隐私控制**：相册、定位、相机、麦克风、通讯录等资源访问必须经过授权
- **ATS 与网络安全**：限制不安全的明文 HTTP 和弱 TLS 连接
- **代码签名与 Entitlements**：保证 App 来源可信，并限制可使用的系统能力
- **代码混淆与字符串保护**：增加静态逆向分析成本
- **反调试、越狱检测、完整性校验**：增加动态分析和篡改成本

这些机制解决的问题不同，不能互相替代。例如 Keychain 解决“敏感数据怎么存”，Sandbox 解决“应用能访问什么”，ATS 解决“网络链路是否安全”，反调试解决“运行时分析成本”。实际项目中需要组合使用。

---

## 2. Keychain

### 2.1 Keychain 是什么

Keychain 是 iOS 提供的系统级安全存储服务，适合保存高敏感数据，例如：

- 登录态 Token、Refresh Token
- 用户密码或密码派生结果
- 客户端证书、私钥
- 支付、风控、设备绑定相关凭据
- App Group 内多个应用共享的安全数据

相比 `UserDefaults`、普通文件、SQLite，Keychain 的优势在于：

- 数据由系统安全服务统一管理
- 可结合设备锁屏状态控制访问时机
- 可配置是否允许备份和迁移到新设备
- 可结合 Access Control 要求 Face ID、Touch ID 或设备密码
- 可以通过 Keychain Sharing 在同一开发团队的 App 之间共享

需要注意：Keychain 保护的是**本地存储安全**，不是接口安全，也不是业务权限安全。服务端仍然必须校验 Token、签名、会话状态和用户权限。

### 2.2 Keychain Item 的核心字段

一个 Keychain 条目通常由多个属性共同定位：

| 字段 | 作用 |
| --- | --- |
| `kSecClass` | 数据类型，例如通用密码、互联网密码、证书、密钥 |
| `kSecAttrService` | 服务名，常用于区分业务模块 |
| `kSecAttrAccount` | 账号名，常用于区分用户或 key |
| `kSecValueData` | 真正保存的数据，通常是 `Data` |
| `kSecAttrAccessible` | 数据可访问时机 |
| `kSecAttrAccessGroup` | Keychain Sharing 分组 |
| `kSecAttrAccessControl` | 生物识别、设备密码等访问控制 |

常见 `kSecClass`：

- `kSecClassGenericPassword`：通用密码，App 保存 Token 最常用
- `kSecClassInternetPassword`：互联网密码，带 server、port、protocol 等属性
- `kSecClassKey`：密钥
- `kSecClassCertificate`：证书
- `kSecClassIdentity`：证书和私钥组合

### 2.3 Accessibility 级别

`kSecAttrAccessible` 决定 Keychain 数据在什么状态下可读：

| 级别 | 含义 | 典型场景 |
| --- | --- | --- |
| `kSecAttrAccessibleWhenUnlocked` | 设备解锁时可访问 | 前台登录态、用户主动操作 |
| `kSecAttrAccessibleAfterFirstUnlock` | 开机后首次解锁后可访问，之后锁屏也可访问 | 后台刷新、静默同步 |
| `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` | 必须设置设备密码，且不能迁移到其他设备 | 高敏感密钥、支付凭据 |
| `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` | 解锁可访问，且不参与备份迁移 | 设备绑定 Token |
| `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` | 首次解锁后可访问，且不迁移 | 后台任务使用的设备级凭据 |

选择原则：

- 普通登录 Token：优先考虑 `kSecAttrAccessibleWhenUnlocked`
- 需要锁屏后后台任务访问：考虑 `kSecAttrAccessibleAfterFirstUnlock`
- 不希望迁移到新设备：使用 `ThisDeviceOnly`
- 高敏感数据：优先考虑 `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly`

### 2.4 Swift 使用示例：保存、读取、删除 Token

```swift
import Foundation
import Security

enum KeychainStore {
    static let service = "com.example.ioser.auth"

    static func saveToken(_ token: String, account: String) throws {
        let data = Data(token.utf8)

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account
        ]

        let attributes: [String: Any] = [
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleWhenUnlockedThisDeviceOnly
        ]

        let status = SecItemUpdate(query as CFDictionary, attributes as CFDictionary)

        if status == errSecItemNotFound {
            var addQuery = query
            addQuery.merge(attributes) { _, new in new }

            let addStatus = SecItemAdd(addQuery as CFDictionary, nil)
            guard addStatus == errSecSuccess else {
                throw KeychainError.unhandledStatus(addStatus)
            }
            return
        }

        guard status == errSecSuccess else {
            throw KeychainError.unhandledStatus(status)
        }
    }

    static func readToken(account: String) throws -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        if status == errSecItemNotFound {
            return nil
        }

        guard status == errSecSuccess else {
            throw KeychainError.unhandledStatus(status)
        }

        guard
            let data = result as? Data,
            let token = String(data: data, encoding: .utf8)
        else {
            throw KeychainError.invalidData
        }

        return token
    }

    static func deleteToken(account: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: service,
            kSecAttrAccount as String: account
        ]

        let status = SecItemDelete(query as CFDictionary)
        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unhandledStatus(status)
        }
    }
}

enum KeychainError: Error {
    case invalidData
    case unhandledStatus(OSStatus)
}
```

这个示例体现了几个关键点：

- 使用 `service + account` 定位一条数据
- 保存前先 `SecItemUpdate`，不存在时再 `SecItemAdd`
- 读取时使用 `kSecReturnData` 和 `kSecMatchLimitOne`
- 删除时兼容 `errSecItemNotFound`
- Token 使用 `ThisDeviceOnly`，避免迁移到其他设备

### 2.5 使用 Face ID / Touch ID 保护 Keychain 数据

对极敏感数据，可以要求读取时必须通过生物识别或设备密码：

```swift
import LocalAuthentication
import Security

func saveSecret(_ secret: Data) throws {
    var error: Unmanaged<CFError>?

    guard let accessControl = SecAccessControlCreateWithFlags(
        nil,
        kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly,
        [.biometryCurrentSet],
        &error
    ) else {
        throw error!.takeRetainedValue() as Error
    }

    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrService as String: "com.example.ioser.secure",
        kSecAttrAccount as String: "private-key",
        kSecValueData as String: secret,
        kSecAttrAccessControl as String: accessControl
    ]

    SecItemDelete(query as CFDictionary)
    let status = SecItemAdd(query as CFDictionary, nil)
    guard status == errSecSuccess else {
        throw KeychainError.unhandledStatus(status)
    }
}
```

常见 Access Control 选项：

- `.biometryAny`：任意已录入生物识别可访问
- `.biometryCurrentSet`：当前生物识别集合可访问，新增或删除面容后失效
- `.userPresence`：生物识别或设备密码验证用户存在
- `.devicePasscode`：必须使用设备密码

使用建议：

- 钱包、支付、私钥类数据优先使用 `.biometryCurrentSet`
- 不要把生物识别当作服务端认证，只能当作本地解锁条件
- 生物识别失败、用户取消、系统锁定都要设计降级路径

### 2.6 Keychain Sharing

Keychain Sharing 允许同一 Team ID 下的多个 App 共享 Keychain 数据。典型场景：

- 主 App 和扩展共享登录态
- 同一公司多个 App 共享账号体系
- App 与 Watch App 共享部分凭据

使用方式：

1. 在 Xcode 的 `Signing & Capabilities` 中开启 `Keychain Sharing`
2. 配置 Access Group，例如 `TEAMID.com.example.shared`
3. 查询字典中增加 `kSecAttrAccessGroup`

```swift
let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrService as String: "com.example.shared.auth",
    kSecAttrAccount as String: "current-user",
    kSecAttrAccessGroup as String: "TEAMID.com.example.shared",
    kSecReturnData as String: true
]
```

注意点：

- Access Group 必须与 Entitlements 匹配
- 模拟器、真机、不同签名环境可能表现不同
- 共享登录态要考虑账号退出时所有 App 的清理策略

### 2.7 Keychain 常见误区

- **误区 1：Keychain 中的数据卸载 App 一定会删除。**  
  不一定。Keychain 数据可能在卸载后保留，具体受系统、访问组和安装状态影响。业务上不能依赖卸载自动清理。

- **误区 2：Token 放进 Keychain 就绝对安全。**  
  不是。越狱、运行时 Hook、内存抓取、调试注入仍可能拿到运行中的 Token。

- **误区 3：Keychain 可以保存客户端密钥并保证永不泄露。**  
  不能。客户端内置密钥最终会在本地出现，只能提高提取成本，不能作为绝对可信根。

- **误区 4：所有敏感数据都应该放 Keychain。**  
  Keychain 适合小块敏感数据，不适合保存大量业务数据、大文件或频繁读写的缓存。

---

## 3. Data Protection：本地文件保护

### 3.1 Data Protection 是什么

Data Protection 是 iOS 针对文件系统提供的数据保护机制。它通过文件保护等级控制文件在设备锁定、解锁、首次解锁后的可访问性。

Keychain 保护的是小块敏感数据，Data Protection 保护的是 App 沙盒中的文件。两者经常配合使用：

- Token、私钥：放 Keychain
- 加密数据库、离线文件、用户文档：使用 Data Protection
- 大量敏感业务数据：业务层加密后落盘，并设置合适的文件保护等级

### 3.2 常见文件保护等级

| 保护等级 | 含义 | 典型场景 |
| --- | --- | --- |
| `.complete` | 设备锁定后不可访问 | 高敏感文件 |
| `.completeUnlessOpen` | 锁屏后已打开文件仍可继续访问 | 长时间写入文件 |
| `.completeUntilFirstUserAuthentication` | 首次解锁后可访问 | 后台任务、数据库 |
| `.none` | 不受锁屏保护 | 非敏感缓存 |

### 3.3 Swift 使用示例

```swift
import Foundation

func writeProtectedFile(data: Data, fileName: String) throws {
    let documentsURL = try FileManager.default.url(
        for: .documentDirectory,
        in: .userDomainMask,
        appropriateFor: nil,
        create: true
    )

    let fileURL = documentsURL.appendingPathComponent(fileName)
    try data.write(to: fileURL, options: [.atomic, .completeFileProtection])
}
```

也可以对已有文件设置属性：

```swift
try FileManager.default.setAttributes(
    [.protectionKey: FileProtectionType.complete],
    ofItemAtPath: fileURL.path
)
```

### 3.4 数据库场景

如果使用 SQLite、Core Data、Realm 等本地数据库，要关注数据库文件、WAL 文件、SHM 文件是否都设置了合适的保护等级。

```swift
let databaseURL = documentsURL.appendingPathComponent("secure.sqlite")
try FileManager.default.setAttributes(
    [.protectionKey: FileProtectionType.completeUntilFirstUserAuthentication],
    ofItemAtPath: databaseURL.path
)
```

注意点：

- 数据库可能生成 `*.sqlite-wal` 和 `*.sqlite-shm`
- App 首次启动且设备未解锁时，受保护文件可能无法读取
- 后台任务需要访问文件时，不能盲目使用 `.complete`
- 敏感数据库最好再叠加 SQLCipher 或业务层字段加密

---

## 4. App Sandbox

### 4.1 App Sandbox 是什么

App Sandbox 是 iOS 的应用隔离机制。每个 App 默认运行在独立容器中，不能随意访问：

- 其他 App 的文件
- 系统敏感目录
- 未授权的相册、相机、定位、麦克风、通讯录等资源
- 其他 App 的进程和内存

Sandbox 的核心价值是把 App 限制在自己的权限边界内，防止一个 App 越权读取或破坏其他 App 的数据。

### 4.2 沙盒目录结构

常见目录如下：

```text
App Container
├── Documents
├── Library
│   ├── Caches
│   ├── Preferences
│   └── Application Support
└── tmp
```

各目录用途：

| 目录 | 用途 | 是否适合敏感数据 |
| --- | --- | --- |
| `Documents` | 用户生成或需要长期保留的数据 | 可以，但要加文件保护或加密 |
| `Library/Application Support` | App 运行所需持久化数据 | 可以，但要加文件保护或加密 |
| `Library/Caches` | 可重新生成的缓存 | 不适合保存敏感数据 |
| `Library/Preferences` | `UserDefaults` 数据 | 不适合保存 Token、密码 |
| `tmp` | 临时文件，系统可能清理 | 不适合保存敏感数据 |

### 4.3 获取沙盒路径示例

```swift
let documentsURL = FileManager.default.urls(
    for: .documentDirectory,
    in: .userDomainMask
).first!

let cachesURL = FileManager.default.urls(
    for: .cachesDirectory,
    in: .userDomainMask
).first!

let temporaryURL = FileManager.default.temporaryDirectory
```

### 4.4 App Group 共享容器

默认情况下，不同 App 或 Extension 的沙盒互相隔离。如果主 App 和 Widget、Share Extension、Notification Service Extension 需要共享数据，可以使用 App Group。

```swift
let groupURL = FileManager.default.containerURL(
    forSecurityApplicationGroupIdentifier: "group.com.example.ioser"
)
```

使用 App Group 时要注意：

- 共享容器扩大了数据可访问范围
- 主 App 和 Extension 都可能读写同一份数据，要处理并发和一致性
- 敏感数据仍然需要加密或设置文件保护
- 退出登录时要清理共享容器中的账号数据

### 4.5 沙盒和权限模型的关系

Sandbox 解决“默认不能访问”的边界问题，权限模型解决“访问敏感资源前必须获得用户授权”的问题。

例如：

- App 不能直接读取系统相册目录，这是 Sandbox 的隔离
- App 调用 `PHPhotoLibrary` 前需要用户授权，这是隐私权限控制
- App 不能直接读取其他 App 的数据库，这是 Sandbox 的隔离
- App Extension 只能访问 App Group 容器，这是 Entitlements 和 Sandbox 共同限制

### 4.6 开发注意点

- 不要把 Token、密码、身份证号等敏感信息写入 `UserDefaults`
- 缓存目录中的文件不应视为安全存储
- 分享、导出、日志、崩溃上报前要做脱敏
- 临时文件使用后及时删除
- App Group 共享数据要有明确的数据所有权和清理策略
- Debug 菜单、日志文件、抓包开关不能进入生产环境

---

## 5. 权限与隐私保护

### 5.1 iOS 权限机制

iOS 对敏感资源使用显式授权机制。常见权限包括：

- 定位：`NSLocationWhenInUseUsageDescription`
- 相机：`NSCameraUsageDescription`
- 麦克风：`NSMicrophoneUsageDescription`
- 相册：`NSPhotoLibraryUsageDescription`
- 通讯录：`NSContactsUsageDescription`
- 蓝牙：`NSBluetoothAlwaysUsageDescription`
- 日历：`NSCalendarsUsageDescription`
- 面容 ID：`NSFaceIDUsageDescription`

如果访问相关 API 但没有在 `Info.plist` 中配置 Usage Description，App 通常会崩溃。

### 5.2 权限请求示例：相机

```swift
import AVFoundation

func requestCameraPermission(completion: @escaping (Bool) -> Void) {
    switch AVCaptureDevice.authorizationStatus(for: .video) {
    case .authorized:
        completion(true)
    case .notDetermined:
        AVCaptureDevice.requestAccess(for: .video) { granted in
            DispatchQueue.main.async {
                completion(granted)
            }
        }
    case .denied, .restricted:
        completion(false)
    @unknown default:
        completion(false)
    }
}
```

### 5.3 权限设计建议

- 在用户真正触发相关功能时再请求权限，不要启动即请求一堆权限
- 权限弹窗文案要解释业务价值，避免模糊描述
- 用户拒绝后不要反复弹窗，应提供设置入口和降级能力
- 对“只允许一次”“仅添加照片”“精确定位关闭”等状态做兼容
- 权限不是安全边界的全部，服务端仍要校验业务访问权限

---

## 6. ATS 与网络安全

### 6.1 ATS 是什么

ATS（App Transport Security）是 iOS 对网络连接安全性的系统约束，默认要求 App 使用 HTTPS，并限制弱加密算法、低版本 TLS、不安全证书等风险。

ATS 的目标是避免以下问题：

- 明文 HTTP 被窃听或篡改
- 使用过期 TLS 协议
- 使用弱加密套件
- 忽略证书校验导致中间人攻击

### 6.2 不建议全局关闭 ATS

`Info.plist` 中全局关闭 ATS 的配置如下，但不建议在生产环境使用：

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

更合理的方式是只对确实需要兼容的域名单独例外，并推动服务端升级 HTTPS。

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSExceptionDomains</key>
    <dict>
        <key>legacy.example.com</key>
        <dict>
            <key>NSExceptionAllowsInsecureHTTPLoads</key>
            <true/>
            <key>NSIncludesSubdomains</key>
            <true/>
        </dict>
    </dict>
</dict>
```

### 6.3 证书校验与 Pinning

HTTPS 默认会进行证书链校验，但高风险业务可能还会做证书绑定（Certificate Pinning 或 Public Key Pinning），限制客户端只信任指定证书或公钥。

证书绑定可以对抗部分中间人攻击，但有维护成本：

- 证书过期或更换时需要提前兼容
- Pinning 配置错误可能导致全量用户无法访问
- 应支持主备证书或公钥
- 不应忽略系统默认的证书链校验

### 6.4 URLSession 证书校验示例

下面是简化版公钥 Pinning 思路，实际项目要处理多证书、证书轮换和错误上报：

```swift
final class PinningDelegate: NSObject, URLSessionDelegate {
    private let pinnedPublicKeyHash: String

    init(pinnedPublicKeyHash: String) {
        self.pinnedPublicKeyHash = pinnedPublicKeyHash
    }

    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard
            challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
            let serverTrust = challenge.protectionSpace.serverTrust
        else {
            completionHandler(.performDefaultHandling, nil)
            return
        }

        var error: CFError?
        guard SecTrustEvaluateWithError(serverTrust, &error) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        // 真实项目中应提取服务端证书公钥并计算 hash，再与本地白名单比较。
        let isPinnedKeyMatched = verifyPinnedPublicKey(serverTrust: serverTrust)

        if isPinnedKeyMatched {
            completionHandler(.useCredential, URLCredential(trust: serverTrust))
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }

    private func verifyPinnedPublicKey(serverTrust: SecTrust) -> Bool {
        // 示例只展示结构，生产代码需要实现公钥提取和 hash 比较。
        return true
    }
}
```

### 6.5 网络安全实践

- 所有业务接口默认使用 HTTPS
- 不要在客户端跳过证书校验
- 不要把接口签名密钥硬编码在客户端
- Token 过期、刷新、撤销要由服务端控制
- 请求日志、抓包日志、崩溃日志不要输出完整 Token
- 高风险接口可以叠加请求签名、时间戳、nonce、防重放校验
- 客户端风控结果只能作为参考，最终决策应在服务端

---

## 7. 代码签名、Entitlements 与系统信任链

### 7.1 代码签名的作用

iOS App 必须经过代码签名才能安装和运行。代码签名的核心作用：

- 验证 App 来源和开发者身份
- 防止 App 安装后被随意篡改
- 绑定 App ID、Team ID 和 Entitlements
- 限制 App 可以使用的系统能力

App 启动和运行时，系统会校验代码签名。如果 Mach-O、资源、Framework 被篡改，签名校验可能失败。

### 7.2 Entitlements 是什么

Entitlements 是 App 被授予的系统能力声明，例如：

- Push Notifications
- Associated Domains
- App Groups
- Keychain Sharing
- iCloud
- HealthKit
- Sign in with Apple

Entitlements 不是普通配置文件，它必须与开发者账号、Provisioning Profile 和签名匹配。

### 7.3 开发中的典型问题

- Keychain Sharing 配置了 Access Group，但 Provisioning Profile 没有对应权限
- App Group ID 在主 App 和 Extension 中不一致
- Associated Domains 配置正确，但服务端 `apple-app-site-association` 不正确
- Debug 签名和 Release 签名使用不同 Team ID，导致 Keychain 读取失败

### 7.4 完整性校验思路

在高风险业务中，App 侧可能会额外检查自身完整性：

- 校验 Bundle 中关键资源是否被替换
- 校验 Mach-O 或关键动态库是否存在异常
- 检查签名信息、Team ID、Bundle ID 是否符合预期
- 检查是否加载了可疑动态库

注意：客户端完整性校验仍然可以被 Hook 或 Patch，只能增加攻击成本，不能作为唯一信任依据。

---

## 8. 代码混淆

### 8.1 代码混淆的目的

代码混淆不是为了“绝对防破解”，而是为了提高静态分析和逆向工程成本。攻击者通常会通过以下信息理解 App：

- 类名、方法名、协议名
- 符号表
- 字符串常量
- 接口路径
- 日志输出
- 业务分支和控制流
- Mach-O 结构和动态库依赖

混淆的目标是减少这些信息的可读性，让攻击者更难快速定位关键逻辑。

### 8.2 Objective-C 的混淆特点

Objective-C 依赖 Runtime，类名、方法名、协议名会以较明显的形式存在于 Mach-O 中。因此 Objective-C 的符号可读性通常较高。

常见加固点：

- 类名、方法名宏替换
- 删除无用符号
- 字符串加密
- 敏感逻辑下沉到 C/C++
- 减少日志和断言信息
- 对关键方法增加完整性校验

示例：简单宏替换思路

```objc
#ifndef DEBUG
#define LoginManager A1B2C3D4
#define requestTokenWithUser B2C3D4E5
#endif
```

这种方式只能降低可读性，不能保护动态调用，因为 Runtime 仍然需要方法名。

### 8.3 Swift 的混淆特点

Swift 相比 Objective-C 暴露的 Runtime 信息少一些，但仍然可能存在：

- 模块名、类型名、泛型信息
- Swift symbol mangling 后的符号
- 字符串常量
- Objective-C 暴露接口，例如 `@objc`、`dynamic`
- 崩溃符号和调试符号

Swift 项目中应减少不必要的 `@objc` 暴露：

```swift
final class TokenSigner {
    func sign(payload: Data) -> String {
        // 核心签名逻辑
        return ""
    }
}
```

如果写成下面这样，动态派发和 Objective-C Runtime 暴露会增加逆向和 Hook 面：

```swift
@objcMembers
class TokenSigner: NSObject {
    dynamic func sign(payload: Data) -> String {
        return ""
    }
}
```

### 8.4 字符串保护

硬编码字符串是逆向分析的重要入口，例如：

- API 域名和路径
- 加密 key
- 业务开关
- 风控参数
- 错误文案中的内部信息

简单字符串混淆示例：

```swift
func decode(_ bytes: [UInt8], key: UInt8) -> String {
    let decoded = bytes.map { $0 ^ key }
    return String(bytes: decoded, encoding: .utf8) ?? ""
}

let apiPath = decode([0x5c, 0x4b, 0x45, 0x43, 0x44], key: 0x23)
```

注意：

- 这类保护只能防止直接 `strings` 扫描
- 解密逻辑和 key 也在客户端，仍可被逆向
- 真正敏感的服务端密钥不应下发到客户端

### 8.5 符号和日志控制

Release 包应关闭不必要的调试信息和日志：

- 不输出完整请求头、Token、手机号、身份证号
- 不输出服务端内部错误栈
- 不把 Debug 菜单和测试入口暴露给生产用户
- 不把 `.dSYM` 打进 App 包
- Strip 不必要符号

Swift 日志示例：

```swift
func logRequest(headers: [String: String]) {
    var sanitizedHeaders = headers
    sanitizedHeaders["Authorization"] = "***"
    sanitizedHeaders["Cookie"] = "***"
    print("request headers: \(sanitizedHeaders)")
}
```

### 8.6 混淆的局限性

- Objective-C Runtime 天然暴露较多元数据
- 动态分析可以绕过静态混淆
- 过度混淆会影响崩溃定位和可维护性
- 三方 SDK 和系统 API 无法完全混淆
- 加固工具可能引入兼容性和审核风险

因此，混淆适合保护核心业务流程、风控逻辑、接口签名入口，但不能替代服务端安全设计。

---

## 9. 反调试、Hook 检测与越狱检测

### 9.1 反调试的目标

反调试主要对抗运行时动态分析。攻击者可能使用：

- LLDB
- Frida
- Cycript
- fishhook
- Method Swizzling
- 越狱插件
- 动态库注入

反调试的目标不是彻底阻止，而是提高分析成本、延缓攻击、发现异常环境并触发风控策略。

### 9.2 使用 `sysctl` 检测调试状态

```swift
import Foundation

func isDebuggerAttached() -> Bool {
    var info = kinfo_proc()
    var size = MemoryLayout<kinfo_proc>.stride

    var name: [Int32] = [
        CTL_KERN,
        KERN_PROC,
        KERN_PROC_PID,
        getpid()
    ]

    let result = name.withUnsafeMutableBufferPointer { pointer -> Int32 in
        sysctl(pointer.baseAddress, u_int(name.count), &info, &size, nil, 0)
    }

    guard result == 0 else {
        return false
    }

    return (info.kp_proc.p_flag & P_TRACED) != 0
}
```

使用方式：

```swift
if isDebuggerAttached() {
    // 生产环境可上报风控事件、限制高风险功能或退出关键流程。
}
```

注意：

- Debug 环境不要启用强制退出
- 单一检测点很容易被绕过
- 检测结果更适合作为风控信号，而不是唯一决策

### 9.3 `ptrace` 反调试

经典做法是调用 `ptrace(PT_DENY_ATTACH, 0, 0, 0)` 阻止调试器附加。

Objective-C 示例：

```objc
#import <dlfcn.h>
#import <sys/types.h>

typedef int (*ptrace_ptr_t)(int _request, pid_t _pid, caddr_t _addr, int _data);

void denyDebuggerAttach(void) {
#if !DEBUG
    void *handle = dlopen(0, RTLD_GLOBAL | RTLD_NOW);
    ptrace_ptr_t ptrace_ptr = dlsym(handle, "ptrace");
    ptrace_ptr(31, 0, 0, 0);
    dlclose(handle);
#endif
}
```

这里用 `dlsym` 获取 `ptrace`，是为了避免符号过于明显。但这仍然可以被 Patch 或 Hook。

### 9.4 检测动态库注入

攻击者常通过注入动态库实现 Hook。可以枚举当前进程加载的动态库：

```swift
import MachO

func hasSuspiciousDynamicLibrary() -> Bool {
    let suspiciousKeywords = [
        "FridaGadget",
        "frida",
        "Substrate",
        "Substitute",
        "cycript"
    ]

    for index in 0..<_dyld_image_count() {
        guard let cName = _dyld_get_image_name(index) else {
            continue
        }

        let imageName = String(cString: cName)
        if suspiciousKeywords.contains(where: { imageName.localizedCaseInsensitiveContains($0) }) {
            return true
        }
    }

    return false
}
```

### 9.5 检测越狱环境

常见越狱检测维度：

- 是否存在越狱相关路径
- 是否能写入系统目录
- 是否能打开非 App Store 环境常见 URL Scheme
- 是否加载了可疑动态库
- Sandbox 是否被破坏

示例：

```swift
import UIKit

func isJailbroken() -> Bool {
#if targetEnvironment(simulator)
    return false
#else
    let suspiciousPaths = [
        "/Applications/Cydia.app",
        "/Library/MobileSubstrate/MobileSubstrate.dylib",
        "/bin/bash",
        "/usr/sbin/sshd",
        "/etc/apt",
        "/private/var/lib/apt/"
    ]

    if suspiciousPaths.contains(where: { FileManager.default.fileExists(atPath: $0) }) {
        return true
    }

    let testPath = "/private/ioser_jailbreak_test.txt"
    do {
        try "test".write(toFile: testPath, atomically: true, encoding: .utf8)
        try FileManager.default.removeItem(atPath: testPath)
        return true
    } catch {
        return false
    }
#endif
}
```

注意：

- 越狱检测极易被绕过
- 新越狱工具路径可能不同
- 误判会影响真实用户
- 最好把结果上报给服务端，由服务端做风险决策

### 9.6 Hook 检测思路

Objective-C Method Swizzling 可以替换方法实现。检测思路包括：

- 检查关键方法 IMP 是否落在预期 Mach-O 范围内
- 检查关键 C 函数地址是否被重绑定
- 检查类、方法列表是否出现异常
- 对关键函数做多点调用链校验

示例：获取方法实现地址

```objc
#import <objc/runtime.h>

IMP currentImplementation(Class cls, SEL selector) {
    Method method = class_getInstanceMethod(cls, selector);
    return method_getImplementation(method);
}
```

这类检测要谨慎，因为系统、三方 SDK、AOP 框架也可能合法使用 Swizzling。

### 9.7 反调试的工程策略

推荐策略：

- 多点检测，不要集中在启动入口
- 检测逻辑分散在关键业务流程中
- Debug、TestFlight、Release 使用不同策略
- 检测结果上报服务端，由服务端结合账号、设备、行为做决策
- 对高风险操作增加二次验证，而不是简单闪退

不推荐：

- 单纯依赖越狱检测决定安全性
- 检测到异常立即崩溃，导致误伤用户和影响排查
- 把所有反调试逻辑写在一个函数中，容易被一次性 Patch

---

## 10. 敏感数据与密钥管理

### 10.1 客户端不能保存真正的服务端秘密

一个重要原则：**客户端不是可信环境**。只要数据被放到 App 包、内存或本地存储中，就有被提取的可能。

不应该放在客户端的内容：

- 服务端私钥
- 全局接口签名密钥
- 可直接控制资金或权限的 master key
- 可以绕过服务端鉴权的固定密钥

可以放在客户端但要保护的内容：

- 短期 Access Token
- Refresh Token
- 设备级随机密钥
- 本地加密数据库的派生密钥
- 风控辅助参数

### 10.2 推荐的 Token 策略

- Access Token 有较短有效期
- Refresh Token 可撤销、可轮换
- 退出登录时清理 Keychain 和本地缓存
- 服务端记录 Token 版本、设备 ID、登录态状态
- 发现异常环境时降低 Token 权限或要求重新认证

### 10.3 本地加密策略

对高敏感本地数据，推荐分层保护：

1. 生成随机数据密钥
2. 用数据密钥加密业务数据
3. 把数据密钥放入 Keychain
4. Keychain 使用 `ThisDeviceOnly` 或生物识别保护
5. 文件本身设置 Data Protection

简化流程：

```text
业务数据
  ↓ 使用 dataKey 加密
加密后的数据库 / 文件

dataKey
  ↓ 保存到 Keychain
受系统保护的小块密钥材料
```

这样即使攻击者拿到数据库文件，没有 Keychain 中的数据密钥也难以直接解密。

---

## 11. 常见安全问题清单

### 11.1 本地存储问题

- Token 放在 `UserDefaults`
- 敏感数据写入日志文件
- 缓存中保存完整用户隐私数据
- SQLite 明文保存身份证号、银行卡号
- 临时文件导出后未删除
- App Group 共享容器退出登录后未清理

### 11.2 网络安全问题

- 使用 HTTP 明文接口
- 全局关闭 ATS
- 忽略证书校验
- 把接口签名密钥硬编码在客户端
- 请求日志打印完整 Header
- 缺少时间戳、nonce、防重放机制

### 11.3 运行时安全问题

- Debug 菜单进入 Release 包
- 测试接口、Mock 开关未关闭
- 越狱、Hook、调试环境没有任何风控
- 核心业务判断完全依赖客户端
- 敏感方法名和字符串过于明显

### 11.4 隐私合规问题

- 启动即请求多个权限
- 未说明权限用途
- 未经同意采集设备信息
- 超范围采集通讯录、相册、定位
- 数据上报未脱敏
- 缺少用户注销和数据删除能力

---

## 12. 一套较完整的安全落地方案

以“登录态和敏感用户数据保护”为例，可以这样设计：

1. 登录成功后服务端返回短期 Access Token 和可轮换 Refresh Token。
2. Token 保存到 Keychain，使用 `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`。
3. 用户敏感资料写入本地数据库前先做字段级加密。
4. 数据库文件设置 `FileProtectionType.completeUntilFirstUserAuthentication`。
5. 所有接口使用 HTTPS，不全局关闭 ATS。
6. 高风险接口使用请求签名、时间戳、nonce 和服务端防重放。
7. Release 包关闭敏感日志，接口日志脱敏。
8. 对核心风控逻辑做字符串保护、符号控制和轻量混淆。
9. 检测调试、越狱、注入动态库等异常信号并上报服务端。
10. 服务端结合账号、设备、环境、行为判断是否要求二次验证。
11. 退出登录时清理 Keychain、数据库、缓存、App Group 共享数据。

这个方案的核心思想是：客户端只做防护和信号采集，最终信任判断放在服务端。

---

## 13. 面试回答思路

如果面试中被问到“iOS 常见安全机制有哪些”，可以按下面结构回答：

1. **系统层安全**：iOS 有代码签名、沙盒、权限控制、Data Protection、Keychain 等基础机制。
2. **数据存储安全**：Token、密码、私钥放 Keychain；文件和数据库结合 Data Protection 或业务层加密；不要放 `UserDefaults`。
3. **网络安全**：使用 HTTPS 和 ATS，不能随意关闭证书校验；高风险业务可以做证书绑定、请求签名、防重放。
4. **应用加固**：通过混淆、字符串加密、符号裁剪、反调试、越狱检测、完整性校验提高逆向和篡改成本。
5. **安全边界认知**：客户端不是可信环境，所有关键权限、交易、风控最终必须由服务端校验。

更完整的回答示例：

> iOS 安全是分层的。系统层面有代码签名保证 App 来源可信，Sandbox 隔离应用数据，权限系统限制相册、相机、定位等资源访问，Keychain 和 Data Protection 保护本地敏感数据。应用层面，开发者需要避免把 Token 放到 UserDefaults，网络请求使用 HTTPS 和 ATS，高风险场景可以做证书绑定、请求签名、防重放。对于逆向风险，可以通过代码混淆、字符串加密、符号裁剪、反调试、越狱检测和完整性校验提高攻击成本。但这些都不能证明客户端绝对可信，最终的身份、权限和资金类操作必须由服务端做校验。

---

## 14. 总结

iOS 安全机制可以理解为一套组合防线：

- **Keychain** 负责小块敏感数据的安全存储
- **Data Protection** 负责本地文件在锁屏状态下的保护
- **App Sandbox** 负责应用间数据和资源隔离
- **权限系统** 负责敏感系统资源访问控制
- **ATS 和 HTTPS** 负责网络传输安全
- **代码签名和 Entitlements** 负责 App 来源、完整性和能力边界
- **混淆和反调试** 负责提高逆向、Hook、篡改成本
- **服务端校验** 负责最终可信判断

实际项目中，安全设计的重点不是追求“客户端绝对安全”，而是通过分层防护降低泄露概率、提高攻击成本、缩小影响范围，并把关键决策放在服务端完成。
