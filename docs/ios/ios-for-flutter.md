# iOS 开发指南（Flutter 开发者视角）

面向 Flutter / Android 开发者的 iOS 打包、签名、上架核心知识。iOS 的签名体系远比 Android 复杂，理解其中的概念关系是解决 99% 打包问题的关键。

**与 Android 的核心对比：**

| | iOS | Android |
|---|---|---|
| 签名工具 | Xcode / Keychain | keytool / jarsigner |
| 签名文件 | Certificate + Provisioning Profile | .keystore 文件 |
| 发布渠道 | App Store / TestFlight | Google Play / 第三方 |
| 测试分发 | TestFlight（最多 10,000 人） | 直接安装 APK |
| 应用 ID | Bundle ID | Application ID |
| 开发者账号费用 | $99/年（个人/公司） | $25 一次性 |
| 审核周期 | 1-3 天 | 数小时至 1 天 |

---

## 核心概念

iOS 签名涉及三个相互绑定的概念，缺一不可：

```
Apple Developer Account
    └── Certificate（证书）        # 证明"你是谁"
    └── App ID / Bundle ID         # 证明"这是什么应用"
    └── Device（设备 UDID）        # 证明"装在哪台设备上"
         └── Provisioning Profile  # 将以上三者绑定在一起
```

### Certificate（证书）

证书证明代码由你签署，存储在 Mac 的 Keychain 中，包含公钥和私钥。

| 类型 | 用途 |
|---|---|
| Apple Development | 开发调试，安装到开发设备 |
| Apple Distribution | 提交 App Store / TestFlight |

> 证书与 Mac 机器绑定，换电脑需要导出 `.p12` 文件迁移。

### Provisioning Profile（描述文件）

描述文件是一个 `.mobileprovision` 文件，将 **证书 + Bundle ID + 设备** 打包绑定，安装到设备时系统验证此文件。

| 类型 | 用途 | 设备限制 |
|---|---|---|
| Development | 真机调试 | 最多 100 台注册设备 |
| Ad Hoc | 测试分发 | 最多 100 台注册设备 |
| App Store | 上架 / TestFlight | 无限制 |

### Bundle ID

iOS 的应用唯一标识，对应 Android 的 `applicationId`。

```
com.yourcompany.yourapp
```

在 Flutter 项目中位于：`ios/Runner.xcodeproj` → General → Bundle Identifier

---

## Apple Developer 账号

### 账号类型

| 类型 | 费用 | 适用场景 |
|---|---|---|
| 个人（Individual） | $99/年 | 个人开发者 |
| 公司（Organization） | $99/年 | 需要 D-U-N-S 编号 |
| 企业（Enterprise） | $299/年 | 内部分发，不能上 App Store |

注册地址：[developer.apple.com](https://developer.apple.com)

### 团队角色

| 角色 | 权限 |
|---|---|
| Account Holder | 全部权限，唯一 |
| Admin | 管理证书、设备、描述文件 |
| Developer | 创建开发证书，有限权限 |

---

## 证书管理

### 创建证书

**方式一：Xcode 自动管理（推荐新手）**

Xcode → Preferences → Accounts → 选择团队 → Manage Certificates → 点击 `+`

**方式二：手动创建**

1. 打开 Keychain Access → 证书助手 → 从证书颁发机构请求证书
2. 填写邮件，选择"存储到磁盘"，生成 `.certSigningRequest` 文件
3. 上传到 [developer.apple.com](https://developer.apple.com) → Certificates → 创建证书

### 导出证书（换电脑 / CI 使用）

```bash
# Keychain Access 中右键证书 → 导出 → 保存为 .p12
# 设置密码，妥善保管
```

在 CI/CD 中使用 Base64 编码存储：

```bash
base64 -i certificate.p12 | pbcopy  # 复制到剪贴板
```

---

## Provisioning Profile 管理

### 自动管理（Automatic Signing）

Xcode 勾选 **Automatically manage signing** 后会自动创建和更新描述文件，适合大多数场景。

Flutter 项目配置：

```bash
# 用 Xcode 打开 iOS 项目
open ios/Runner.xcworkspace
```

Signing & Capabilities → 勾选 Automatically manage signing → 选择 Team

### 手动管理（Manual Signing）

适用于 CI/CD 或需要精确控制的场景。

1. 在 [developer.apple.com](https://developer.apple.com) → Profiles → 创建描述文件
2. 选择类型（App Store / Ad Hoc / Development）
3. 绑定 Bundle ID、证书、设备
4. 下载 `.mobileprovision` 文件，双击安装到 Xcode

---

## Flutter 项目 iOS 配置

### 关键文件

```
ios/
├── Runner.xcworkspace          # 用这个打开，不要用 .xcodeproj
├── Runner.xcodeproj/
│   └── project.pbxproj         # 签名配置存储在这里
├── Runner/
│   ├── Info.plist              # 应用权限声明
│   └── GoogleService-Info.plist # Firebase 配置（如果有）
└── Podfile                     # CocoaPods 依赖
```

### Info.plist 常用权限

```xml
<!-- 相机 -->
<key>NSCameraUsageDescription</key>
<string>需要访问相机以拍摄照片</string>

<!-- 相册 -->
<key>NSPhotoLibraryUsageDescription</key>
<string>需要访问相册以选择图片</string>

<!-- 位置 -->
<key>NSLocationWhenInUseUsageDescription</key>
<string>需要获取位置以提供导航服务</string>

<!-- 麦克风 -->
<key>NSMicrophoneUsageDescription</key>
<string>需要访问麦克风以录制音频</string>
```

> Flutter 插件通常会在 README 中说明需要添加哪些权限。

### 最低系统版本

```ruby
# ios/Podfile
platform :ios, '13.0'
```

```
# Xcode → Runner → General → Minimum Deployments
iOS 13.0
```

---

## 打包（Archive）

### 命令行打包（推荐用于 CI）

```bash
# 1. 构建 Flutter
flutter build ios --release

# 2. 用 xcodebuild 打包（自动签名）
xcodebuild -workspace ios/Runner.xcworkspace \
  -scheme Runner \
  -configuration Release \
  -archivePath build/Runner.xcarchive \
  archive

# 3. 导出 IPA（App Store）
xcodebuild -exportArchive \
  -archivePath build/Runner.xcarchive \
  -exportPath build/Runner.ipa \
  -exportOptionsPlist ios/ExportOptions.plist
```

### ExportOptions.plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>method</key>
    <string>app-store</string>  <!-- app-store | ad-hoc | development -->
    <key>teamID</key>
    <string>XXXXXXXXXX</string>  <!-- 10位 Team ID，在 developer.apple.com 查看 -->
    <key>uploadSymbols</key>
    <true/>
    <key>compileBitcode</key>
    <false/>
</dict>
</plist>
```

### Xcode 图形界面打包

1. Xcode → Product → Archive
2. 等待构建完成，Organizer 窗口自动打开
3. Distribute App → App Store Connect → Upload

---

## TestFlight 分发

TestFlight 是 Apple 官方的测试分发平台，上传方式与正式版完全相同。

### 内部测试 vs 外部测试

| | 内部测试 | 外部测试 |
|---|---|---|
| 人数 | 最多 100 人 | 最多 10,000 人 |
| 资格 | 团队成员（需 Apple ID 加入团队） | 任何人（邮件邀请） |
| 审核 | 无需审核 | 需 Beta App Review |
| 有效期 | 90 天 | 90 天 |

### 上传 IPA

**方式一：Transporter（Mac App）**

下载 Transporter → 拖入 `.ipa` → 上传

**方式二：命令行**

```bash
xcrun altool --upload-app \
  --type ios \
  --file build/Runner.ipa/Runner.ipa \
  --apiKey YOUR_API_KEY \
  --apiIssuer YOUR_ISSUER_ID
```

**方式三：Xcode Organizer**

Archive 后 → Distribute App → App Store Connect → Upload

---

## App Store 上架流程

### 1. App Store Connect 配置

登录 [appstoreconnect.apple.com](https://appstoreconnect.apple.com)：

- 新建 App → 填写 Bundle ID、SKU、名称
- 填写截图（iPhone 6.5"、iPad 12.9" 必填）
- 填写描述、关键词、隐私政策 URL
- 选择分级、类别

### 2. 截图尺寸要求

| 设备 | 尺寸 | 必填 |
|---|---|---|
| iPhone 6.7"（15 Pro Max） | 1290 × 2796 | 推荐 |
| iPhone 6.5"（14 Plus） | 1242 × 2688 | 必填 |
| iPhone 5.5"（8 Plus） | 1242 × 2208 | 可选 |
| iPad Pro 12.9" | 2048 × 2732 | 必填（如支持 iPad） |

### 3. 提交审核

App Store Connect → 选择构建版本 → 填写审核信息 → 提交审核

审核要点：
- 所有请求的权限必须实际使用
- 不能包含私有 API
- 应用功能需完整，无占位内容
- 内购项目需正确配置

### 4. 版本号规范

Flutter 的 `pubspec.yaml`：

```yaml
version: 1.2.3+45
#        └─┬─┘ └┤
#    CFBundleShortVersionString  （App Store 显示的版本）
#              CFBundleVersion   （构建号，每次上传必须递增）
```

---

## CI/CD 签名方案

### Fastlane Match（推荐团队使用）

Match 将证书和描述文件加密存储在 Git 仓库，团队成员和 CI 共享同一套签名文件。

```bash
gem install fastlane
fastlane match init      # 初始化，填写 Git 仓库地址
fastlane match appstore  # 同步 App Store 证书
fastlane match development  # 同步开发证书
```

`Matchfile`：

```ruby
git_url("https://github.com/yourorg/certificates")
app_identifier("com.yourcompany.yourapp")
username("your@email.com")
```

### GitHub Actions 完整示例

```yaml
- name: Install certificates
  env:
    P12_BASE64: ${{ secrets.IOS_P12_BASE64 }}
    P12_PASSWORD: ${{ secrets.IOS_P12_PASSWORD }}
    PROVISION_BASE64: ${{ secrets.IOS_PROVISION_BASE64 }}
  run: |
    # 创建临时 Keychain
    security create-keychain -p "" build.keychain
    security default-keychain -s build.keychain
    security unlock-keychain -p "" build.keychain

    # 导入证书
    echo $P12_BASE64 | base64 --decode > certificate.p12
    security import certificate.p12 -k build.keychain \
      -P $P12_PASSWORD -T /usr/bin/codesign

    # 安装描述文件
    mkdir -p ~/Library/MobileDevice/Provisioning\ Profiles
    echo $PROVISION_BASE64 | base64 --decode \
      > ~/Library/MobileDevice/Provisioning\ Profiles/profile.mobileprovision
```

---

## 常见问题

### No signing certificate found

```
No signing certificate "iOS Distribution" found
```

原因：当前 Mac 没有对应证书，或证书已过期。

解决：Xcode → Preferences → Accounts → Manage Certificates → 重新创建

---

### Provisioning profile doesn't include signing certificate

原因：描述文件绑定的证书与当前证书不匹配。

解决：重新生成描述文件，或在 developer.apple.com 将当前证书加入描述文件。

---

### The app bundle is invalid（上传时）

常见原因：
- `CFBundleVersion`（构建号）未递增
- 包含模拟器架构（`arm64` 与 `x86_64` 混合）

```bash
# 检查 IPA 架构
lipo -info Payload/Runner.app/Runner
```

---

### Pod install 失败

```bash
cd ios
rm -rf Pods Podfile.lock
pod install --repo-update
```

---

### Flutter 版本升级后签名失效

```bash
flutter clean
cd ios && pod install
```

重新用 Xcode 打开，检查 Signing & Capabilities 配置是否还在。
