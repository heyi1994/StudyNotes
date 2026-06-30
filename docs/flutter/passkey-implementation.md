# 通行密钥（Passkey）

> 适用平台：Android 原生、iOS 原生、Flutter
> 协议基础：FIDO2 / WebAuthn / CTAP2

---

## 1. 通行密钥基础概念

### 1.1 什么是通行密钥

通行密钥（Passkey）是基于 **FIDO2 / WebAuthn** 标准的无密码认证凭据。它使用**非对称密钥对**替代传统密码：

- **私钥**：保存在用户设备的安全硬件中（Android 的 TEE/StrongBox、iOS 的 Secure Enclave），永不离开设备，并可通过云端钥匙串（iCloud Keychain / Google Password Manager）端到端加密同步。
- **公钥**：注册时上传至业务服务器（依赖方 RP）保存。

认证时，服务器下发挑战值（challenge），设备用私钥签名，服务器用公钥验签。私钥从不传输，因此**天然抗钓鱼、抗重放、抗撞库**。

### 1.2 核心术语

| 术语                        | 含义                                                               |
| --------------------------- | ------------------------------------------------------------------ |
| **RP (Relying Party)**      | 依赖方，即你的应用/网站，由 `rpId`（域名）标识                     |
| **Authenticator**           | 认证器，即生成和保管私钥的实体（平台认证器或漫游认证器如安全密钥） |
| **Credential**              | 凭据，一对密钥 + `credentialId`                                    |
| **Challenge**               | 服务端生成的随机数，防重放，必须一次性使用                         |
| **Attestation**             | 注册阶段，认证器对自身可信度的证明                                 |
| **Assertion**               | 认证阶段，用私钥对挑战的签名响应                                   |
| **User Verification (UV)**  | 用户验证（生物识别/PIN），证明"是本人"                             |
| **Discoverable Credential** | 可发现凭据（旧称 resident key），支持"用户名优先/无用户名"登录     |

### 1.3 两大核心流程

- **注册（Registration / Attestation）**：创建一个新的通行密钥并绑定到用户账号。
- **认证（Authentication / Assertion）**：使用已有通行密钥登录。

---

## 2. 整体架构与流程

### 2.1 注册流程时序

```
客户端                          业务服务器                      认证器(设备)
  │                                │                              │
  │ 1. 请求注册选项                 │                              │
  │ ───────────────────────────►  │                              │
  │                                │ 生成 challenge、user.id      │
  │ 2. 返回 PublicKeyCredential-   │                              │
  │    CreationOptions (JSON)      │                              │
  │ ◄───────────────────────────  │                              │
  │                                │                              │
  │ 3. 调用平台 API 创建凭据         │                              │
  │ ──────────────────────────────────────────────────────────► │
  │                                │              生物识别/PIN验证 │
  │                                │              生成密钥对        │
  │ 4. 返回 attestation 响应        │                              │
  │ ◄────────────────────────────────────────────────────────── │
  │                                │                              │
  │ 5. 提交 attestation 给服务器    │                              │
  │ ───────────────────────────►  │ 验签、存储 publicKey +       │
  │                                │ credentialId                 │
  │ 6. 注册成功                     │                              │
  │ ◄───────────────────────────  │                              │
```

### 2.2 认证流程时序

```
客户端                          业务服务器                      认证器(设备)
  │ 1. 请求认证选项                 │                              │
  │ ───────────────────────────►  │ 生成 challenge               │
  │ 2. 返回 PublicKeyCredential-   │                              │
  │    RequestOptions (JSON)       │                              │
  │ ◄───────────────────────────  │                              │
  │ 3. 调用平台 API 获取断言         │                              │
  │ ──────────────────────────────────────────────────────────► │
  │                                │              生物识别验证      │
  │                                │              私钥签名 challenge│
  │ 4. 返回 assertion 响应          │                              │
  │ ◄────────────────────────────────────────────────────────── │
  │ 5. 提交 assertion               │ 用公钥验签、校验            │
  │ ───────────────────────────►  │ challenge/origin/signCount   │
  │ 6. 登录成功，下发 session/token │                              │
  │ ◄───────────────────────────  │                              │
```

---

## 3. 服务端准备工作（通用）

三个平台共享同一套服务端，强烈建议服务端实现严格遵循 WebAuthn 规范。推荐使用成熟库：

- Java/Kotlin：[`webauthn4j`](https://github.com/webauthn4j/webauthn4j)、`java-webauthn-server` (Yubico)
- Node.js：[`@simplewebauthn/server`](https://simplewebauthn.dev/)

### 3.1 关联域名文件（最关键！）

通行密钥要求 App 与某个域名（`rpId`）建立可信关联，否则系统会拒绝创建/使用凭据。

#### Android — Digital Asset Links

在你的网站根路径部署 `https://example.com/.well-known/assetlinks.json`：

```json
[
  {
    "relation": ["delegate_permission/common.get_login_creds"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.example.app",
      "sha256_cert_fingerprints": [
        "AB:CD:EF:...:01" // 应用签名证书的 SHA-256 指纹
      ]
    }
  }
]
```

获取签名指纹：

```bash
keytool -list -v -keystore my-release.keystore -alias my-alias
# 或对已上架应用，使用 Play Console 中的 App signing 证书指纹
```

#### iOS — Apple App Site Association (AASA)

在 `https://example.com/.well-known/apple-app-site-association` 部署（注意：无扩展名、`Content-Type: application/json`、不可重定向）：

```json
{
  "webcredentials": {
    "apps": ["TEAMID.com.example.app"]
  }
}
```

> 注意：`webcredentials` 是通行密钥所需的关键字段（不同于 universal links 的 `applinks`）。

### 3.2 注册选项示例（服务器返回）

```json
{
  "challenge": "base64url-随机32字节",
  "rp": { "id": "example.com", "name": "Example" },
  "user": {
    "id": "base64url-用户唯一ID",
    "name": "user@example.com",
    "displayName": "User Name"
  },
  "pubKeyCredParams": [
    { "type": "public-key", "alg": -7 }, // ES256
    { "type": "public-key", "alg": -257 } // RS256
  ],
  "timeout": 1800000,
  "attestation": "none",
  "excludeCredentials": [],
  "authenticatorSelection": {
    "authenticatorAttachment": "platform",
    "residentKey": "required",
    "requireResidentKey": true,
    "userVerification": "required"
  }
}
```

### 3.3 认证选项示例（服务器返回）

```json
{
  "challenge": "base64url-随机32字节",
  "rpId": "example.com",
  "timeout": 1800000,
  "userVerification": "required",
  "allowCredentials": [] // 空数组 = 可发现凭据，支持账号选择器
}
```

> **重要**：`challenge`、`user.id`、`credentialId` 等二进制字段在 JSON 中均使用 **Base64URL（无填充）** 编码。服务端与客户端务必统一编码方式。

---

## 4. Android 原生接入

Android 通行密钥通过 **Credential Manager（Jetpack）** 实现，是 Google 官方推荐的统一凭据 API。

### 4.1 环境要求

- 设备：Android 9 (API 28) 及以上（部分能力 API 34+ 更完善）
- 依赖：

```kotlin
// build.gradle.kts (app)
dependencies {
    implementation("androidx.credentials:credentials:1.3.0")
    implementation("androidx.credentials:credentials-play-services-auth:1.3.0")
}
```

### 4.2 注册（创建通行密钥）

```kotlin
import androidx.credentials.CredentialManager
import androidx.credentials.CreatePublicKeyCredentialRequest
import androidx.credentials.CreatePublicKeyCredentialResponse
import androidx.credentials.exceptions.CreateCredentialException

suspend fun registerPasskey(activity: Activity) {
    val credentialManager = CredentialManager.create(activity)

    // 1. 从服务器获取注册选项 JSON（见 3.2）
    val requestJson: String = fetchCreationOptionsFromServer()

    // 2. 构造请求
    val createRequest = CreatePublicKeyCredentialRequest(
        requestJson = requestJson,
        preferImmediatelyAvailableCredentials = false
    )

    try {
        // 3. 弹出系统 UI，用户生物识别确认
        val result = credentialManager.createCredential(
            context = activity,
            request = createRequest
        ) as CreatePublicKeyCredentialResponse

        // 4. result.registrationResponseJson 即 attestation 响应
        val attestationJson = result.registrationResponseJson

        // 5. 提交服务器验签存储
        sendAttestationToServer(attestationJson)

    } catch (e: CreateCredentialException) {
        handleCreateError(e)  // 见 4.4 错误处理
    }
}
```

`registrationResponseJson` 结构（符合 WebAuthn `PublicKeyCredential`）：

```json
{
  "id": "credentialId-base64url",
  "rawId": "credentialId-base64url",
  "type": "public-key",
  "response": {
    "clientDataJSON": "base64url",
    "attestationObject": "base64url",
    "transports": ["internal", "hybrid"]
  },
  "authenticatorAttachment": "platform"
}
```

### 4.3 认证（使用通行密钥登录）

```kotlin
import androidx.credentials.GetCredentialRequest
import androidx.credentials.GetPublicKeyCredentialOption
import androidx.credentials.PublicKeyCredential
import androidx.credentials.exceptions.GetCredentialException

suspend fun authenticateWithPasskey(activity: Activity) {
    val credentialManager = CredentialManager.create(activity)

    // 1. 服务器返回认证选项 JSON（见 3.3）
    val requestJson: String = fetchRequestOptionsFromServer()

    val getOption = GetPublicKeyCredentialOption(requestJson = requestJson)
    val getRequest = GetCredentialRequest(listOf(getOption))

    try {
        val result = credentialManager.getCredential(
            context = activity,
            request = getRequest
        )

        val credential = result.credential
        if (credential is PublicKeyCredential) {
            // 2. authenticationResponseJson 即 assertion
            val assertionJson = credential.authenticationResponseJson
            // 3. 提交服务器验签
            sendAssertionToServer(assertionJson)
        }
    } catch (e: GetCredentialException) {
        handleGetError(e)
    }
}
```

### 4.4 错误处理

```kotlin
import androidx.credentials.exceptions.*
import androidx.credentials.exceptions.publickeycredential.*

fun handleCreateError(e: CreateCredentialException) {
    when (e) {
        is CreatePublicKeyCredentialDomException ->
            // WebAuthn 规范错误，检查 domError（如 InvalidStateError = 已注册）
            Log.e("Passkey", "DOM error: ${e.domError}")
        is CreateCredentialCancellationException ->
            Log.i("Passkey", "用户取消")
        is CreateCredentialInterruptedException ->
            Log.w("Passkey", "可重试")
        is CreateCredentialProviderConfigurationException ->
            Log.e("Passkey", "缺少 credentials-play-services-auth 依赖")
        is CreateCredentialUnknownException ->
            Log.e("Passkey", "未知错误")
        else -> Log.e("Passkey", "其他: ${e.message}")
    }
}
```

### 4.5 Android 注意事项

- **Google Password Manager** 是默认平台提供方，需设备登录 Google 账号且开启同步。
- **条件式 UI（Conditional UI / Autofill）**：在登录页输入框设置 `autofill hints` 并使用 `prepareGetCredential`，可在键盘上方直接展示通行密钥建议。
- `assetlinks.json` 部署后，可能需要等待 Google 缓存刷新（最长数小时）。可用 [Digital Asset Links 测试工具](https://developers.google.com/digital-asset-links/tools/generator) 验证。
- 三星等定制 ROM 可能存在认证器兼容差异，需真机测试。

---

## 5. iOS 原生接入

iOS 通行密钥通过 **Authentication Services 框架**的 `ASAuthorizationPlatformPublicKeyCredentialProvider` 实现。

### 5.1 环境要求

- 系统：iOS 16+（通行密钥），iOS 15 仅支持设备本地密钥
- Xcode：14+
- 能力配置：在 **Signing & Capabilities** 中添加 **Associated Domains**：

```
webcredentials:example.com
```

> 可加 `?mode=developer` 在开发期绕过 AASA 的 CDN 缓存（需在设备"开发者设置"中开启 Associated Domains Development）。

### 5.2 注册（创建通行密钥）

```swift
import AuthenticationServices

class PasskeyManager: NSObject,
    ASAuthorizationControllerDelegate,
    ASAuthorizationControllerPresentationContextProviding {

    let domain = "example.com"

    func registerPasskey(userName: String, userID: Data, challenge: Data) {
        let provider = ASAuthorizationPlatformPublicKeyCredentialProvider(
            relyingPartyIdentifier: domain
        )

        let request = provider.createCredentialRegistrationRequest(
            challenge: challenge,      // 服务器下发，需 base64url 解码为 Data
            name: userName,
            userID: userID             // 服务器下发的 user.id
        )
        request.userVerificationPreference = .required

        let controller = ASAuthorizationController(authorizationRequests: [request])
        controller.delegate = self
        controller.presentationContextProvider = self
        controller.performRequests()
    }

    // 回调：成功
    func authorizationController(
        controller: ASAuthorizationController,
        didCompleteWithAuthorization authorization: ASAuthorization
    ) {
        if let reg = authorization.credential as?
            ASAuthorizationPlatformPublicKeyCredentialRegistration {

            let credentialId = reg.credentialID                 // Data
            let attestationObject = reg.rawAttestationObject    // Data?
            let clientDataJSON = reg.rawClientDataJSON           // Data

            // 转 base64url 后组装成 WebAuthn JSON 提交服务器
            sendAttestationToServer(
                id: credentialId.base64URLEncodedString(),
                attestationObject: attestationObject?.base64URLEncodedString(),
                clientDataJSON: clientDataJSON.base64URLEncodedString()
            )
        }
    }

    func authorizationController(
        controller: ASAuthorizationController,
        didCompleteWithError error: Error
    ) {
        handleError(error)
    }
}
```

### 5.3 认证（使用通行密钥登录）

```swift
func authenticateWithPasskey(challenge: Data) {
    let provider = ASAuthorizationPlatformPublicKeyCredentialProvider(
        relyingPartyIdentifier: domain
    )

    let request = provider.createCredentialAssertionRequest(challenge: challenge)
    request.userVerificationPreference = .required
    // 可选：限定 allowedCredentials
    // request.allowedCredentials = [.init(credentialID: savedCredId)]

    let controller = ASAuthorizationController(authorizationRequests: [request])
    controller.delegate = self
    controller.presentationContextProvider = self
    controller.performRequests()
    // 若做 AutoFill：controller.performAutoFillAssistedRequests()
}

// 在 didCompleteWithAuthorization 中区分类型
func handleAssertion(_ authorization: ASAuthorization) {
    if let assertion = authorization.credential as?
        ASAuthorizationPlatformPublicKeyCredentialAssertion {

        sendAssertionToServer(
            id: assertion.credentialID.base64URLEncodedString(),
            authenticatorData: assertion.rawAuthenticatorData.base64URLEncodedString(),
            signature: assertion.signature.base64URLEncodedString(),
            clientDataJSON: assertion.rawClientDataJSON.base64URLEncodedString(),
            userHandle: assertion.userID.base64URLEncodedString()
        )
    }
}
```

### 5.4 Base64URL 辅助扩展

```swift
extension Data {
    func base64URLEncodedString() -> String {
        base64EncodedString()
            .replacingOccurrences(of: "+", with: "-")
            .replacingOccurrences(of: "/", with: "_")
            .replacingOccurrences(of: "=", with: "")
    }

    init?(base64URLEncoded input: String) {
        var s = input
            .replacingOccurrences(of: "-", with: "+")
            .replacingOccurrences(of: "_", with: "/")
        while s.count % 4 != 0 { s += "=" }
        self.init(base64Encoded: s)
    }
}
```

### 5.5 iOS 注意事项

- 通行密钥默认通过 **iCloud Keychain** 同步，需设备登录 iCloud 且开启钥匙串同步。
- **AutoFill（Conditional UI）**：在 `UITextField` 设置 `.textContentType = .username` 后调用 `performAutoFillAssistedRequests()`，可在键盘上方展示通行密钥。
- AASA 文件修改后，系统使用 Apple CDN 缓存，开发期务必加 `?mode=developer` 并在设置中开启。
- 错误码：`ASAuthorizationError.canceled`（用户取消）、`.failed`、`.invalidResponse`、`.notHandled`。

---

## 6. Flutter 接入

Flutter 不内置通行密钥能力，需借助插件或自建 Platform Channel 桥接原生。

### 6.1 方案选型

| 方案                                                      | 说明                           | 适用场景                     |
| --------------------------------------------------------- | ------------------------------ | ---------------------------- |
| **官方/社区插件**（如 `passkeys`、`webauthn` by Corbado） | 封装好双端原生调用，开箱即用   | 快速接入、标准 WebAuthn 流程 |
| **Platform Channel 自建桥接**                             | 自行调用 §4/§5 原生 API        | 需深度定制、强管控依赖       |
| **WebView + 网页 WebAuthn**                               | 在 WebView 内走浏览器 WebAuthn | 已有成熟 Web 实现，混合 App  |

> 推荐：标准流程用 [`passkeys`](https://pub.dev/packages/passkeys) 插件（Corbado 维护，封装 Android Credential Manager + iOS AuthenticationServices）。

### 6.2 使用 `passkeys` 插件

#### 添加依赖

```yaml
# pubspec.yaml
dependencies:
  passkeys: ^2.x.x
```

#### 平台配置

- **Android**：部署 `assetlinks.json`（§3.1），`minSdkVersion` ≥ 28。
- **iOS**：添加 Associated Domains `webcredentials:example.com`，部署 AASA，iOS 16+。

#### 注册

```dart
import 'package:passkeys/authenticator.dart';
import 'package:passkeys/types.dart';

final authenticator = PasskeyAuthenticator();

Future<void> registerPasskey() async {
  // 1. 服务器获取注册选项
  final options = await api.fetchRegisterOptions();

  // 2. 调用原生创建凭据
  final result = await authenticator.register(
    RegisterRequestType(
      challenge: options.challenge,           // base64url
      relyingParty: RelyingPartyType(
        id: 'example.com',
        name: 'Example',
      ),
      user: UserType(
        id: options.userId,                   // base64url
        name: 'user@example.com',
        displayName: 'User Name',
      ),
      authSelectionType: AuthenticatorSelectionType(
        authenticatorAttachment: 'platform',
        requireResidentKey: true,
        residentKey: 'required',
        userVerification: 'required',
      ),
      pubKeyCredParams: [
        PubKeyCredParamType(type: 'public-key', alg: -7),
        PubKeyCredParamType(type: 'public-key', alg: -257),
      ],
      timeout: 1800000,
      excludeCredentials: [],
      attestation: 'none',
    ),
  );

  // 3. 提交服务器
  await api.completeRegister(
    id: result.id,
    rawId: result.rawId,
    clientDataJSON: result.clientDataJSON,
    attestationObject: result.attestationObject,
  );
}
```

#### 认证

```dart
Future<void> authenticatePasskey() async {
  final options = await api.fetchLoginOptions();

  final result = await authenticator.authenticate(
    AuthenticateRequestType(
      relyingPartyId: 'example.com',
      challenge: options.challenge,
      timeout: 1800000,
      userVerification: 'required',
      allowCredentials: [],       // 空 = 可发现凭据
      mediation: MediationType.Optional,
    ),
  );

  await api.completeLogin(
    id: result.id,
    rawId: result.rawId,
    clientDataJSON: result.clientDataJSON,
    authenticatorData: result.authenticatorData,
    signature: result.signature,
    userHandle: result.userHandle,
  );
}
```

#### 错误处理

```dart
try {
  await registerPasskey();
} on PasskeyAuthCancelledException {
  // 用户取消
} on PasskeyAuthExcludeCredentialsCanMatchError {
  // 设备上已存在该凭据
} on DeviceNotSupportedException {
  // 设备/系统版本不支持
} on PasskeyAuthException catch (e) {
  // 其他
  debugPrint('Passkey error: ${e.message}');
}
```

### 6.3 自建 Platform Channel 桥接（进阶）

当需要完全掌控时，可定义 MethodChannel，Dart 侧发请求，原生侧按 §4 / §5 实现并回传 JSON。

```dart
// Dart 侧
const _channel = MethodChannel('app/passkey');

Future<String> nativeCreatePasskey(String optionsJson) async {
  final res = await _channel.invokeMethod<String>('create', {
    'optionsJson': optionsJson,
  });
  return res!;  // attestation JSON
}
```

```kotlin
// Android 侧（MainActivity 或独立 Plugin）
channel.setMethodCallHandler { call, result ->
    when (call.method) {
        "create" -> {
            val optionsJson = call.argument<String>("optionsJson")!!
            lifecycleScope.launch {
                try {
                    val req = CreatePublicKeyCredentialRequest(optionsJson)
                    val resp = CredentialManager.create(this@MainActivity)
                        .createCredential(this@MainActivity, req)
                        as CreatePublicKeyCredentialResponse
                    result.success(resp.registrationResponseJson)
                } catch (e: Exception) {
                    result.error("PASSKEY_ERR", e.message, null)
                }
            }
        }
    }
}
```

```swift
// iOS 侧 —— 用 §5 的 ASAuthorization 实现，回调里 result(jsonString)
```

### 6.4 Flutter 注意事项

- 插件本质仍依赖各平台关联域名配置，**`assetlinks.json` / AASA 缺一不可**。
- iOS 端需保证调用发生在有 `presentationContextProvider` 的窗口上下文中（插件已处理，自建需注意）。
- WebView 方案中，iOS 的 `WKWebView` 对 WebAuthn 支持有限制，需 iOS 16+ 且配置妥当；优先用原生桥接。

---

## 7. 关键数据结构对照

| 概念         | Android (Credential Manager)       | iOS (AuthenticationServices)          | Flutter (passkeys)         |
| ------------ | ---------------------------------- | ------------------------------------- | -------------------------- |
| 入口对象     | `CredentialManager`                | `ASAuthorizationController`           | `PasskeyAuthenticator`     |
| 注册请求     | `CreatePublicKeyCredentialRequest` | `createCredentialRegistrationRequest` | `RegisterRequestType`      |
| 注册响应     | `registrationResponseJson`         | `...CredentialRegistration`           | `RegisterResponseType`     |
| 认证请求     | `GetPublicKeyCredentialOption`     | `createCredentialAssertionRequest`    | `AuthenticateRequestType`  |
| 认证响应     | `authenticationResponseJson`       | `...CredentialAssertion`              | `AuthenticateResponseType` |
| 输入输出格式 | WebAuthn JSON 字符串               | 原生 `Data` 字段需手动组装 JSON       | 字段化对象（base64url）    |
| 域名关联     | `assetlinks.json`                  | AASA `webcredentials`                 | 两者皆需                   |

> Android 的 Credential Manager 直接吞吐标准 WebAuthn JSON，最省心；iOS 需要自己在 `Data` 与 base64url JSON 间转换。

---

## 8. 常见问题与排查

| 现象                                          | 可能原因                         | 排查                                                                      |
| --------------------------------------------- | -------------------------------- | ------------------------------------------------------------------------- |
| 创建凭据弹窗不出现                            | 关联域名未生效                   | 检查 `assetlinks.json` / AASA 是否可公网访问、Content-Type 正确、无重定向 |
| iOS 报 `notHandled` / 无反应                  | Associated Domains 未配置或缓存  | 加 `?mode=developer`，重装 App                                            |
| Android `ProviderConfigurationException`      | 缺 play-services-auth 依赖       | 添加 `credentials-play-services-auth`                                     |
| `InvalidStateError` / ExcludeCredentials 命中 | 该设备已为此账号注册过           | 走"已有通行密钥"提示，或调整 `excludeCredentials`                         |
| 服务器验签失败                                | challenge/origin/rpIdHash 不匹配 | 核对 base64url 编码、`rpId` 与域名一致、origin 校验规则                   |
| signCount 异常                                | 凭据被克隆或同步设备计数为 0     | 同步凭据 signCount 常为 0，验签逻辑需容忍                                 |
| 用户无账号选择器                              | 未用可发现凭据                   | 注册时 `residentKey: required`，认证时 `allowCredentials: []`             |

调试技巧：

- 用浏览器先验证服务端 WebAuthn 流程（`webauthn.io` 风格自测页）。
- 校验 `clientDataJSON` 中的 `origin`：Android 为 `android:apk-key-hash:...`，iOS/Web 为 `https://example.com`。服务端需对 App origin 做相应校验。

---

## 9. 安全与合规要点

1. **Challenge 一次性**：每次注册/认证生成新随机 challenge（≥16 字节），用后即弃，设置短超时。
2. **服务端必须验签**：客户端返回结果不可信，所有校验（签名、challenge、origin、rpIdHash、UV flag、signCount）在服务端完成。
3. **校验 origin**：白名单方式校验 App / Web origin，防止伪造来源。
4. **userVerification: required**：金融/支付场景强制用户验证（生物识别/PIN）。
5. **attestation 策略**：消费级应用通常 `none` 即可；高安全场景用 `direct` 并校验认证器证书链（注意隐私权衡）。
6. **降级与恢复**：通行密钥应与其他登录方式（密码、邮箱魔法链接、短信）共存，提供账号恢复路径，避免用户锁死。
7. **凭据生命周期**：提供"管理已注册通行密钥"界面，支持列出、命名、撤销凭据。
8. **同步与多设备**：iCloud Keychain / Google Password Manager 会跨设备同步，signCount 可能恒为 0，验签逻辑勿强依赖递增计数。

---

## 附录：参考资料

- WebAuthn Level 2/3 规范：https://www.w3.org/TR/webauthn/
- FIDO Alliance：https://fidoalliance.org/passkeys/
- Android Credential Manager 文档：https://developer.android.com/training/sign-in/passkeys
- Apple 通行密钥文档：https://developer.apple.com/documentation/authenticationservices/public-private_key_authentication
- passkeys (Flutter) 插件：https://pub.dev/packages/passkeys
- SimpleWebAuthn（服务端/前端参考实现）：https://simplewebauthn.dev/

---
