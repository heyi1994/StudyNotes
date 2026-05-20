# Android 17 (API 37) 行为变更文档

> **版本状态**：Beta 3 已于 2026 年 3 月 26 日达到平台稳定，API 接口已锁定。正式版预计 2026 年 6 月发布。
> **文档来源**：[Android Developers 官方文档](https://developer.android.com/about/versions/17)

## 一、影响所有应用的变更

无论 `targetSdkVersion` 为何值，只要运行在 Android 17 设备上即受影响。

---

### 1.1 应用内存限制

**类别**：核心功能 | **影响级别**：高

Android 17 根据设备 RAM 为应用引入内存上限，旨在防止系统因极端内存泄漏而不稳定。

| 应用类型                    | 键数量上限 |
| --------------------------- | ---------- |
| 非系统应用（target API 37） | 50,000     |
| 其他应用                    | 200,000    |
| 系统应用                    | 200,000    |

**检测方式**

```java
ApplicationExitInfo exitInfo = getApplicationContext().getApplicationExitInfo();
String description = exitInfo.getDescription();
if (description != null && description.contains("MemoryLimiter:AnonSwap")) {
    // 应用因内存超限被终止
}
```

**ADB 调试命令**

```bash
am memory-limiter status
am memory-limiter ignore <uid>|none|all
am memory-limiter manual <pid> <limit>|max|none
```

**迁移建议**：遵循[内存最佳实践](https://developer.android.com/topic/performance/memory)，使用 `TRIGGER_TYPE_ANOMALY` 触发堆转储分析。

---

### 1.2 SMS OTP 保护（WebOTP 格式）

**类别**：隐私 | **影响级别**：高

将现有的 SMS Retriever 保护扩展至 WebOTP 格式消息。非接收应用在消息到达后 **3 小时内**无法访问该消息。

**影响行为**：

- `SMS_RECEIVED_ACTION` 广播被延迟
- SMS Provider 数据库查询结果被过滤

**豁免应用**：默认短信处理器、设备配套应用（companion apps）等

**迁移建议**：改用 [SMS Retriever API](https://developers.google.com/identity/sms-retriever) 或 [SMS User Consent API](https://developers.google.com/identity/sms-retriever/user-consent/overview)。

---

### 1.3 跨 Profile 回环流量拦截

**类别**：连接 | **影响级别**：中

运行在 Android 17 上的所有应用，跨 Profile（如工作 Profile 与个人 Profile 之间）的回环流量默认不再被允许。同一 Profile 内的回环流量不受影响。

**迁移建议**：使用标准 IPC 机制进行跨 Profile 通信，并在多 Profile 场景下充分测试。

---

### 1.4 旋转后 IME 可见性不再恢复

**类别**：用户体验 | **影响级别**：中

设备旋转等未处理的配置变更后，系统**不再**自动恢复软键盘的显示状态。

**解决方案一：清单属性**

```xml
<activity
    android:name=".MyActivity"
    android:windowSoftInputMode="stateAlwaysVisible" />
```

**解决方案二：代码显式请求**

```java
InputMethodManager imm = getSystemService(InputMethodManager.class);
imm.showSoftInput(editText, InputMethodManager.SHOW_IMPLICIT);
```

**解决方案三：自行处理配置变更**

```java
@Override
public void onConfigurationChanged(Configuration newConfig) {
    super.onConfigurationChanged(newConfig);
    InputMethodManager imm = getSystemService(InputMethodManager.class);
    imm.showSoftInput(editText, InputMethodManager.SHOW_IMPLICIT);
}
```

---

### 1.5 触摸板指针捕获时发送相对事件

**类别**：输入 | **影响级别**：低

调用 `View.requestPointerCapture()` 后，触摸板现在与鼠标行为一致，发送**相对事件**（而非绝对坐标），并识别滚动手势。

```java
// 默认（相对模式）：触摸板行为等同鼠标
view.requestPointerCapture();

// 绝对模式（需要原始手指坐标时）
view.requestPointerCapture(View.POINTER_CAPTURE_MODE_ABSOLUTE);
```

---

### 1.6 后台音频行为强化

**类别**：媒体 | **影响级别**：高

音频框架对后台应用的音频操作执行生命周期限制，防止意外的后台音频播放。

**限制范围**：音频播放、音频焦点请求、音量变更

**失败行为**：

- 音频播放/音量 API：静默失败（不抛出异常）
- 音频焦点 API：返回 `AUDIOFOCUS_REQUEST_FAILED`

**推荐做法**：在前台服务中执行音频操作。

```java
@Override
public int onStartCommand(Intent intent, int flags, int startId) {
    startForeground(NOTIFICATION_ID, notification);

    AudioManager audioManager = getSystemService(AudioManager.class);
    int result = audioManager.requestAudioFocus(
        focusChangeListener,
        AudioAttributes.USAGE_MEDIA,
        AudioManager.AUDIOFOCUS_GAIN
    );
    return START_STICKY;
}
```

> 另见 [2.5 后台音频强化（更严格）](#25-后台音频强化更严格)——针对 target API 37 有额外要求。

---

### 1.7 蓝牙自主重配对

**类别**：连接 | **影响级别**：低

Android 17 引入系统级自主重配对能力，在蓝牙绑定丢失时**自动**在后台重建连接，无需用户手动进入设置。

**关键变更**：

1. 新增配对上下文 `PAIRING_CONTEXT_SYSTEM_INITIATED`

```java
int pairingContext = intent.getIntExtra(
    BluetoothDevice.EXTRA_PAIRING_CONTEXT, -1);

if (pairingContext == BluetoothDevice.PAIRING_CONTEXT_SYSTEM_INITIATED) {
    // 系统发起的自主重配对，无需用户干预
}
```

2. `ACTION_KEY_MISSING` 广播仅在重配对**失败后**发出，减少误报。

**测试建议**：验证应用能优雅处理绑定过渡状态，包括重配对成功与失败两种场景。

---

### 1.8 Keystore 密钥数量限制

**类别**：安全 | **影响级别**：中

Android 17 对 Android Keystore 中存储的密钥数量设置上限，防止滥用。

| 应用类型                    | 上限    |
| --------------------------- | ------- |
| 非系统应用（target API 37） | 50,000  |
| 其他应用                    | 200,000 |
| 系统应用                    | 200,000 |

**错误处理**

```java
try {
    // 创建 Keystore 密钥
} catch (KeyStoreException e) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
        if (e.getNumericErrorCode() == KeyStoreException.ERROR_TOO_MANY_KEYS) {
            // 处理密钥数量超限
        }
    }
}
```

---

### 1.9 隐式 URI 授权限制（提前警告）

**类别**：安全 | **影响级别**：中（预警）

当前系统会在 `ACTION_SEND`、`ACTION_SEND_MULTIPLE`、`ACTION_IMAGE_CAPTURE` 等 Intent 中**自动**授予 URI 读写权限。**Android 18** 将移除此行为，建议现在就显式授权。

```java
Intent intent = new Intent(Intent.ACTION_SEND);
intent.setData(uri);
// 显式声明权限，避免依赖隐式授权
intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
intent.addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION);
startActivity(intent);
```

---

### 1.10 明文流量（usesCleartextTraffic）弃用预告

**类别**：安全 | **影响级别**：低（预警）

`usesCleartextTraffic` 清单属性将在未来版本中弃用，建议迁移至 Network Security Configuration。

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">example.com</domain>
    </domain-config>
</network-security-config>
```

---

## 二、仅影响 targetSdkVersion=37 应用的变更

仅当应用 `targetSdkVersion` 设置为 37 (Android 17) 时才受以下影响。

---

### 2.1 MessageQueue 无锁实现

**类别**：核心功能 | **影响级别**：高

Android 17 对 `android.os.MessageQueue` 进行了重大重写，采用无锁（lock-free）实现，减少丢帧、提升性能。**但若应用通过反射访问 MessageQueue 的私有字段或方法，将会崩溃。**

**影响场景**：通过反射操作 `mMessages`、`mNextBarrierToken` 等私有成员的代码。

**迁移建议**：

- 停止使用反射访问 `MessageQueue` 私有 API
- 改用公开 API：`MessageQueue.addIdleHandler()`、`Looper.myQueue()` 等

详见：[MessageQueue 行为变更指南](https://developer.android.com/about/versions/17/changes/messagequeue)

---

### 2.2 static final 字段不可通过反射/JNI 修改

**类别**：核心功能 | **影响级别**：高

应用不能再通过反射或 JNI 修改 `static final` 字段。

| 修改方式                  | 结果                          |
| ------------------------- | ----------------------------- |
| Java 反射 `Field.set()`   | 抛出 `IllegalAccessException` |
| JNI `SetStaticXxxField()` | 应用直接崩溃                  |

**迁移建议**：停止使用此类 hack，改用可变状态容器或依赖注入。

---

### 2.3 本地网络访问权限强制要求

**类别**：隐私 | **影响级别**：高

访问本地网络（局域网设备、mDNS 等）需要声明并在运行时请求新权限 `ACCESS_LOCAL_NETWORK`，该权限属于 `NEARBY_DEVICES` 权限组。

**清单声明**

```xml
<uses-permission android:name="android.permission.ACCESS_LOCAL_NETWORK" />
```

**运行时请求**

```java
if (ContextCompat.checkSelfPermission(this,
        Manifest.permission.ACCESS_LOCAL_NETWORK)
        != PackageManager.PERMISSION_GRANTED) {
    ActivityCompat.requestPermissions(this,
        new String[]{Manifest.permission.ACCESS_LOCAL_NETWORK},
        REQUEST_CODE_LOCAL_NETWORK);
}
```

**替代方案**：使用系统提供的隐私保护设备选择器（device picker），无需申请权限。

---

### 2.4 标准 SMS 消息 OTP 三小时延迟

**类别**：隐私 | **影响级别**：高

不符合 SMS Retriever 或 WebOTP 格式的标准 SMS 消息，非接收应用需等待 **3 小时**后才能通过程序读取。

**影响范围**：

- `SMS_RECEIVED_ACTION` 广播延迟下发
- SMS Provider 数据库查询结果被过滤

**豁免条件**：默认短信应用、已注册的配套设备应用

**迁移建议**：使用 [SMS Retriever API](https://developers.google.com/identity/sms-retriever) 或 [SMS User Consent API](https://developers.google.com/identity/sms-retriever/user-consent/overview) 规避延迟。

---

### 2.5 后台音频强化（更严格）

**类别**：媒体 | **影响级别**：高

针对 target API 37 的应用，后台音频操作须同时满足以下条件：

1. 应用必须运行**前台服务**；且
2. 满足以下**至少一项**：
   - 前台服务具有 while-in-use (WIU) 能力
   - 应用拥有 Exact Alarm 权限，且使用 `USAGE_ALARM` 音频流

```java
// 推荐：使用 MediaSessionService 管理媒体播放
// 确保前台服务类型正确声明
<service
    android:name=".MyMediaService"
    android:foregroundServiceType="mediaPlayback" />
```

详见：[后台音频强化指南](https://developer.android.com/about/versions/17/changes/bg-audio)

---

### 2.6 大屏幕方向/可调整大小限制完全忽略

**类别**：设备形态 | **影响级别**：高

在最小宽度 ≥ 600dp 的大屏幕设备（平板、折叠屏）上，以下清单属性将被系统**完全忽略**：

- `screenOrientation`
- `resizeableActivity`
- `minAspectRatio`
- `maxAspectRatio`

> **注意**：Android 16 允许通过特定方式 opt-out；Android 17 中 opt-out 能力已被移除，不再支持任何例外。

**迁移建议**：应用必须支持自适应布局，参考[大屏幕适配指南](https://developer.android.com/guide/topics/large-screens/large-screen-ui-guide)。

详见：[方向和可调整大小限制忽略](https://developer.android.com/about/versions/17/changes/ff-restrictions-ignored)

---

### 2.7 蓝牙 RFCOMM Socket read() 行为一致性

**类别**：连接 | **影响级别**：低

`BluetoothSocket` 的 `InputStream.read()` 方法在连接断开或套接字关闭时，现在统一返回 `-1`（与 LE CoC 套接字及标准 Java `InputStream` 保持一致）。

**之前行为**：抛出 `IOException`
**现在行为**：返回 `-1`

**迁移代码**

```java
byte[] buffer = new byte[1024];
int bytesRead;
while ((bytesRead = inputStream.read(buffer)) != -1) {
    // 处理数据
    processData(buffer, bytesRead);
}
// 循环退出 = 连接已断开
handleDisconnection();
```

---

### 2.8 Activity 安全（BAL 硬化）

**类别**：安全 | **影响级别**：高

后台 Activity 启动（BAL）限制扩展至 `IntentSender`。

| 状态                                              | 变更                               |
| ------------------------------------------------- | ---------------------------------- |
| `MODE_BACKGROUND_ACTIVITY_START_ALLOWED`          | **已弃用**                         |
| `MODE_BACKGROUND_ACTIVITY_START_ALLOW_IF_VISIBLE` | **推荐替代**（仅调用方可见时允许） |

**迁移建议**：

- 使用 Strict Mode 检测违规
- 升级 Android Studio 以获取最新 Lint 检查
- 仅在用户可见的上下文中启动 Activity

---

### 2.9 证书透明度（CT）默认启用

**类别**：安全 | **影响级别**：中

证书透明度验证现在**默认启用**（Android 16 需要主动 opt-in）。所有 TLS 连接必须提供有效的 CT 证书。

**配置方式**：如需针对特定域名调整，使用 Network Security Configuration：

```xml
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">internal.example.com</domain>
        <!-- 内部 CA 通常不在 CT 日志中，可视情况配置 -->
    </domain-config>
</network-security-config>
```

---

### 2.10 原生库动态加载必须只读

**类别**：安全 | **影响级别**：中

通过 `System.load()` 动态加载的原生库文件必须标记为**只读**（只读文件系统权限）。

**违规后果**：抛出 `UnsatisfiedLinkError`

**迁移建议**：

- 避免在运行时动态生成并加载原生库
- 确保动态加载的 `.so` 文件权限为只读
- 优先从 APK 内置资源加载

---

### 2.11 联系人 CP2 PII 字段访问限制

**类别**：隐私 | **影响级别**：中

无 `READ_CONTACTS` 权限时，通过 `ContactsContract.Data` 视图访问以下字段将被拒绝：

- `ACCOUNT_NAME`
- `ACCOUNT_TYPE`
- `ACCOUNT_TYPE_AND_DATA_SET`

同时，`ContactsContract.Data` 查询启用 `StrictColumns` 和 `StrictGrammar` 检查，不兼容的查询将抛出异常。

**迁移方案**：通过 `ContactsContract.RawContacts` 使用 `RAW_CONTACT_ID` 联接获取账户信息，并声明 `READ_CONTACTS` 权限。

---

### 2.12 加密客户端 Hello（ECH）默认启用

**类别**：隐私 | **影响级别**：中

TLS 握手中的 SNI（服务器名称指示）现在默认通过 ECH（Encrypted Client Hello）加密，防止网络观察者识别目标域名。

**配置**：在 Network Security Configuration 中使用 `<domainEncryption>` 元素进行精细控制。

详见：[ECH 配置说明](https://developer.android.com/privacy-and-security/security-config#EncryptedClientHelloSummary)

---

### 2.13 物理键盘输入密码时默认隐藏字符

**类别**：隐私 | **影响级别**：低

使用外接物理键盘输入密码时，系统应用 `show_passwords_physical` 设置，**默认隐藏**所有密码字符（与触屏设备使用 `show_passwords_touch` 设置相对应）。

---

## 三、优先级汇总表

### 影响所有应用

| 变更                    | 类别     | 影响级别 | 迁移复杂度 |
| ----------------------- | -------- | -------- | ---------- |
| 应用内存限制            | 核心     | 高       | 中         |
| SMS OTP 保护（WebOTP）  | 隐私     | 高       | 中         |
| 后台音频行为强化        | 媒体     | 高       | 中         |
| Keystore 密钥数量限制   | 安全     | 中       | 低         |
| 跨 Profile 回环流量拦截 | 连接     | 中       | 低         |
| 旋转后 IME 可见性       | 用户体验 | 中       | 低         |
| 蓝牙自主重配对          | 连接     | 低       | 低         |
| 触摸板指针捕获          | 输入     | 低       | 低         |
| 隐式 URI 授权（预警）   | 安全     | 中       | 低         |
| 明文流量弃用（预警）    | 安全     | 低       | 低         |

### 仅影响 targetSdkVersion=37

| 变更                    | 类别   | 影响级别 | 迁移复杂度 |
| ----------------------- | ------ | -------- | ---------- |
| 本地网络访问权限        | 隐私   | **高**   | 中         |
| 标准 SMS OTP 三小时延迟 | 隐私   | **高**   | 高         |
| 后台音频（更严格要求）  | 媒体   | **高**   | 高         |
| 大屏幕方向限制忽略      | 大屏幕 | **高**   | 高         |
| Activity BAL 硬化       | 安全   | **高**   | 中         |
| MessageQueue 无锁实现   | 核心   | 高       | 中         |
| static final 反射禁止   | 核心   | 高       | 低         |
| 证书透明度默认启用      | 安全   | 中       | 低         |
| CP2 PII 字段限制        | 隐私   | 中       | 中         |
| 原生库只读加载          | 安全   | 中       | 低         |
| ECH 默认启用            | 隐私   | 中       | 低         |
| 蓝牙 RFCOMM read()      | 连接   | 低       | 低         |
| 物理键盘密码隐藏        | 隐私   | 低       | 低         |

---

## 四、迁移建议与测试

### 测试环境准备

1. 下载 [Android 17 SDK](https://developer.android.com/about/versions/17/setup-sdk)，安装 API 37 模拟器镜像
2. 使用支持 Android 17 的 Pixel 设备（Pixel 6 及以上）
3. 升级 Android Studio 至最新版本获取最新 Lint 规则

### 分阶段迁移策略

**第一阶段（立即处理，所有 targetSdk）**

- [ ] 审查后台音频相关代码，确保通过前台服务操作
- [ ] 排查 SMS 读取逻辑，迁移至 SMS Retriever API
- [ ] 使用 Memory Profiler 分析内存使用，避免超限

**第二阶段（升级 targetSdk=37 前处理）**

- [ ] 搜索并消除对 `MessageQueue` 私有字段的反射访问
- [ ] 搜索并消除对 `static final` 字段的反射/JNI 修改
- [ ] 添加 `ACCESS_LOCAL_NETWORK` 权限声明及运行时请求逻辑
- [ ] 适配大屏幕自适应布局（特别是锁定方向的界面）
- [ ] 审查蓝牙 RFCOMM 读取循环，改为检查返回值 `-1`
- [ ] 替换弃用的 BAL 常量为新 API

**第三阶段（完整验证）**

- [ ] 在大屏幕（sw≥600dp）设备/模拟器上测试所有界面
- [ ] 使用 Strict Mode 检测 BAL 违规
- [ ] 验证蓝牙场景下的重配对体验
- [ ] 在多 Profile 环境中验证网络连接行为
- [ ] 检查 `ApplicationExitInfo` 中是否存在内存超限终止记录

### 常用调试命令

```bash
# 检测内存超限退出
adb shell dumpsys activity processes | grep MemoryLimiter

# 测试本地网络权限
adb shell pm grant <package> android.permission.ACCESS_LOCAL_NETWORK

# 检查应用后台音频状态
adb shell dumpsys audio | grep <package>

# 查看应用退出原因
adb shell dumpsys activity exit-info <package>
```
