# Fastlane

Fastlane 是专为 iOS 和 Android 移动应用自动化设计的开源工具集，用 Ruby 编写。它将截图生成、签名管理、打包构建、上传发布等繁琐的手动操作统一封装成可重复执行的自动化流程，是移动端 CI/CD 的事实标准。

**核心概念：**

| 概念 | 说明 |
|------|------|
| **Fastfile** | 核心配置文件，用 Ruby DSL 定义所有 lane |
| **lane** | 一条自动化流水线，由多个 action 顺序组成 |
| **action** | 内置功能单元，如 `gym`（构建）、`pilot`（上传 TestFlight）|
| **plugin** | 社区扩展 action，通过 `fastlane-plugin-xxx` 安装 |
| **Appfile** | 存储 App ID、Team ID 等应用元信息 |
| **Matchfile** | `match` 工具的配置，管理证书和 Provisioning Profile |
| **Deliverfile** | `deliver` 工具的配置，管理 App Store 元数据 |
| **Gymfile** | `gym` 工具的配置，管理构建参数 |
| **Screengrabfile** | Android 截图自动化配置 |
| **Snapfile** | iOS 截图自动化配置 |

---

## 安装

### 前置要求

| 平台 | 要求 |
|------|------|
| iOS | macOS + Xcode（最新稳定版）|
| Android | JDK 8+、Android SDK |
| 通用 | Ruby 2.6+（建议用 rbenv/rvm 管理）|

### 安装 Fastlane

```shell
# 方式一：Homebrew（推荐 macOS）
brew install fastlane

# 方式二：RubyGems
gem install fastlane

# 方式三：Bundler（推荐团队协作，锁定版本）
# 在项目根目录创建 Gemfile
bundle init
echo 'gem "fastlane"' >> Gemfile
bundle install

# 验证安装
fastlane --version
fastlane actions          # 查看所有内置 action
```

### 使用 Bundler 管理版本（推荐）

```ruby
# Gemfile
source "https://rubygems.org"

gem "fastlane", "~> 2.220"
gem "cocoapods"                        # iOS 依赖管理

# 插件
gem "fastlane-plugin-firebase_app_distribution"
gem "fastlane-plugin-versioning"
```

```shell
bundle install
bundle exec fastlane <lane>            # 通过 Bundler 运行，保证版本一致
```

---

## 初始化项目

```shell
# 进入项目根目录
cd your-project

# iOS 初始化
fastlane init

# Android 初始化
fastlane init

# 交互式向导，选择用途：
# 1. Automate screenshots
# 2. Automate beta distribution to TestFlight
# 3. Automate App Store distribution
# 4. Manual setup
```

初始化后生成的目录结构：

```
your-project/
├── fastlane/
│   ├── Fastfile          # 核心：lane 定义
│   ├── Appfile           # 应用信息（Bundle ID、Apple ID 等）
│   ├── Matchfile         # 证书管理配置（iOS）
│   ├── Deliverfile       # App Store 上传配置（iOS）
│   ├── Gymfile           # 构建配置（iOS）
│   ├── Screengrabfile    # Android 截图配置
│   ├── Snapfile          # iOS 截图配置
│   └── metadata/         # App Store 文字描述、截图等
│       ├── zh-Hans/
│       └── en-US/
├── Gemfile
└── Gemfile.lock
```

---

## Fastfile 语法详解

### 基本结构

```ruby
# fastlane/Fastfile

# 设置最低 Fastlane 版本要求
fastlane_version "2.200.0"

# 默认平台（ios / android）
default_platform(:ios)

platform :ios do
  # ── 生命周期钩子 ────────────────────────
  before_all do |lane|
    ensure_git_status_clean              # 确保工作目录干净
    git_pull                             # 拉取最新代码
  end

  after_all do |lane|
    notify_team(lane: lane)              # 自定义通知
  end

  error do |lane, exception|
    slack(
      message: "Lane #{lane} 失败：#{exception.message}",
      success: false
    )
  end

  # ── Lane 定义 ────────────────────────────
  desc "运行所有测试"
  lane :test do
    run_tests(scheme: "MyApp")
  end

  desc "打包并上传到 TestFlight"
  lane :beta do
    increment_build_number
    build_app(scheme: "MyApp")
    upload_to_testflight(skip_waiting_for_build_processing: true)
  end

  desc "发布到 App Store"
  lane :release do
    capture_screenshots
    build_app(scheme: "MyApp")
    upload_to_app_store(
      force:                    true,
      submit_for_review:        true,
      automatic_release:        false
    )
  end
end

platform :android do
  desc "打包 Release APK"
  lane :build do
    gradle(task: "clean assembleRelease")
  end

  desc "上传到 Google Play 内测"
  lane :beta do
    gradle(task: "bundle", build_type: "Release")
    upload_to_play_store(track: "internal")
  end
end
```

### lane 基础语法

```ruby
# 带参数的 lane
lane :deploy do |options|
  env     = options[:env]     || "staging"
  version = options[:version] || get_version_number

  puts "部署 #{version} 到 #{env}"

  build_app(
    scheme:        "MyApp-#{env.capitalize}",
    configuration: env == "prod" ? "Release" : "Staging"
  )
end

# 调用带参数的 lane
# fastlane deploy env:prod version:2.0.0

# lane 调用另一个 lane
lane :ci do
  test
  beta
end

# 私有 lane（不暴露在命令行，只能被其他 lane 调用）
private_lane :notify_team do |options|
  slack(message: "Lane #{options[:lane]} 完成")
end
```

### 变量与环境

```ruby
# 读取环境变量
api_key = ENV["APP_STORE_CONNECT_API_KEY"]

# .env 文件（fastlane 自动加载 fastlane/.env）
# fastlane/.env
# SLACK_URL=https://hooks.slack.com/...
# MATCH_PASSWORD=your_password

# 指定 .env 文件运行
# fastlane beta --env staging

# fastlane/.env.staging
# APP_IDENTIFIER=com.example.app.staging
```

---

## iOS 核心 Actions

### 代码签名（match）

`match` 是 Fastlane 推荐的证书管理方案（Code Signing Guide 的实现），将证书和 Provisioning Profile 存储到 Git 仓库或 S3/GCS，团队共享同一套签名配置。

```ruby
# fastlane/Matchfile
git_url("git@github.com:your-org/certificates.git")
storage_mode("git")              # git | s3 | google_cloud
type("development")              # development | adhoc | appstore | enterprise
app_identifier(["com.example.app", "com.example.app.extension"])
username("apple@example.com")
```

```shell
# 初始化证书仓库（首次）
fastlane match init

# 生成并存储证书
fastlane match development
fastlane match adhoc
fastlane match appstore

# 团队成员同步证书
fastlane match development --readonly

# 撤销并重新生成
fastlane match nuke development   # 危险：撤销所有 development 证书
fastlane match development
```

```ruby
# 在 Fastfile 中使用 match
lane :beta do
  match(
    type:        "adhoc",
    readonly:    is_ci,           # CI 上只读，不自动生成
    app_identifier: "com.example.app"
  )
  build_app(scheme: "MyApp")
  distribute_via_firebase
end
```

### 构建应用（gym / build_app）

```ruby
# fastlane/Gymfile（全局默认配置）
scheme("MyApp")
workspace("MyApp.xcworkspace")
output_directory("./build")
output_name("MyApp")
clean(true)
export_method("app-store")          # app-store | ad-hoc | development | enterprise

# Fastfile 中使用
lane :build_release do
  build_app(
    scheme:              "MyApp",
    workspace:           "MyApp.xcworkspace",
    configuration:       "Release",
    export_method:       "app-store",
    output_directory:    "./build",
    output_name:         "MyApp.ipa",
    clean:               true,
    include_bitcode:     false,
    export_options: {
      provisioningProfiles: {
        "com.example.app"           => "match AppStore com.example.app",
        "com.example.app.extension" => "match AppStore com.example.app.extension"
      }
    }
  )
end
```

### 版本号管理

```ruby
# 读取当前版本
version = get_version_number(xcodeproj: "MyApp.xcodeproj", target: "MyApp")
build   = get_build_number(xcodeproj: "MyApp.xcodeproj")

# 递增 Build Number（常用于 CI）
increment_build_number(
  build_number: ENV["CI_BUILD_NUMBER"] || Time.now.strftime("%Y%m%d%H%M")
)

# 递增版本号
increment_version_number(bump_type: "patch")   # 1.0.0 → 1.0.1
increment_version_number(bump_type: "minor")   # 1.0.0 → 1.1.0
increment_version_number(bump_type: "major")   # 1.0.0 → 2.0.0
increment_version_number(version_number: "2.0.0")

# 使用 agvtool（更精准）
increment_build_number(build_number: latest_testflight_build_number + 1)
```

### 测试（scan / run_tests）

```ruby
lane :test do
  run_tests(
    workspace:   "MyApp.xcworkspace",
    scheme:      "MyAppTests",
    devices:     ["iPhone 15", "iPad Pro (12.9-inch)"],
    clean:       true,
    code_coverage: true,
    output_directory: "fastlane/test_output",
    output_types:   "html,junit",   # 生成 HTML 和 JUnit 报告
    fail_build:  true
  )
end
```

### 上传 TestFlight（pilot / upload_to_testflight）

```ruby
lane :beta do
  # 使用 App Store Connect API Key（推荐，不依赖 2FA）
  api_key = app_store_connect_api_key(
    key_id:      ENV["ASC_KEY_ID"],
    issuer_id:   ENV["ASC_ISSUER_ID"],
    key_content: ENV["ASC_KEY_CONTENT"],      # Base64 编码的 .p8 内容
    in_house:    false
  )

  upload_to_testflight(
    api_key:                           api_key,
    ipa:                               "./build/MyApp.ipa",
    changelog:                         "修复若干 Bug",
    beta_app_review_info: {
      contact_email:      "dev@example.com",
      contact_first_name: "Dev",
      contact_last_name:  "Team",
      contact_phone:      "+86-000-0000-0000"
    },
    groups:                            ["内测组", "QA 组"],
    skip_waiting_for_build_processing: true    # 不等待处理完成（CI 推荐）
  )
end
```

### 上传 App Store（deliver / upload_to_app_store）

```ruby
lane :release do
  api_key = app_store_connect_api_key(
    key_id:      ENV["ASC_KEY_ID"],
    issuer_id:   ENV["ASC_ISSUER_ID"],
    key_content: ENV["ASC_KEY_CONTENT"]
  )

  upload_to_app_store(
    api_key:              api_key,
    ipa:                  "./build/MyApp.ipa",
    app_version:          get_version_number,

    # 元数据（也可以放在 fastlane/metadata/ 目录下的文件中）
    name: {
      "zh-Hans" => "我的应用",
      "en-US"   => "My App"
    },
    release_notes: {
      "zh-Hans" => "· 修复已知问题\n· 性能优化",
      "en-US"   => "· Bug fixes\n· Performance improvements"
    },

    submit_for_review:     true,
    automatic_release:     false,
    force:                 true,    # 不需要人工确认
    skip_screenshots:      true,    # 跳过截图上传
    skip_metadata:         false,

    # 审核信息
    submission_information: {
      add_id_info_uses_idfa: false,
      export_compliance_uses_encryption: false
    }
  )
end
```

### 自动截图（snapshot / capture_screenshots）

```ruby
# fastlane/Snapfile
devices([
  "iPhone 15 Pro Max",
  "iPhone SE (3rd generation)",
  "iPad Pro (12.9-inch) (6th generation)"
])
languages(["zh-Hans", "en-US"])
scheme("MyAppUITests")
output_directory("./fastlane/screenshots")
clear_previous_screenshots(true)
headless(true)
```

```ruby
lane :screenshots do
  capture_screenshots(scheme: "MyAppUITests")

  # 生成美化的截图框架（需要 frameit）
  frame_screenshots(
    white: true,
    path:  "./fastlane/screenshots"
  )
end
```

### App Store Connect API Key 配置

```shell
# 在 App Store Connect → Users and Access → Keys 生成 .p8 文件
# 记录 Key ID 和 Issuer ID

# 将 .p8 内容转为 Base64 存入 CI 环境变量
base64 -i AuthKey_XXXXXXXX.p8 | tr -d '\n'
```

---

## Android 核心 Actions

### 构建应用（gradle）

```ruby
# fastlane/Fastfile (Android)
platform :android do
  desc "构建 Debug APK"
  lane :build_debug do
    gradle(
      task:       "assemble",
      build_type: "Debug",
      print_command: true
    )
  end

  desc "构建 Release AAB"
  lane :build_release do
    gradle(
      task:        "bundle",
      build_type:  "Release",
      properties: {
        "android.injected.signing.store.file"     => ENV["KEYSTORE_PATH"],
        "android.injected.signing.store.password" => ENV["KEYSTORE_PASS"],
        "android.injected.signing.key.alias"      => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password"   => ENV["KEY_PASS"]
      }
    )
  end

  desc "执行 Android 单元测试"
  lane :test do
    gradle(task: "test")
  end
end
```

### 版本号管理（Android）

```ruby
# 读取 versionName 和 versionCode
version_name = get_android_version_name    # 需要 fastlane-plugin-versioning
version_code = get_android_version_code

# 递增 versionCode
increment_version_code(
  gradle_file_path: "app/build.gradle",
  version_code:     ENV["CI_BUILD_NUMBER"].to_i
)

# 设置 versionName
increment_version_name(
  gradle_file_path: "app/build.gradle",
  bump_type:        "patch"               # major | minor | patch
)

# 也可以直接用 Android Gradle 属性
lane :set_version do |options|
  android_set_version_name(
    version_name:     options[:version],
    gradle_file_path: "app/build.gradle"
  )
  android_set_version_code(
    version_code:     options[:build_number].to_i,
    gradle_file_path: "app/build.gradle"
  )
end
```

### 代码签名（Android）

```ruby
# 方式一：通过 Gradle 属性传入（推荐）
lane :sign do
  gradle(
    task:       "bundle",
    build_type: "Release",
    properties: {
      "android.injected.signing.store.file"     => ENV["KEYSTORE_FILE"],
      "android.injected.signing.store.password" => ENV["STORE_PASSWORD"],
      "android.injected.signing.key.alias"      => ENV["KEY_ALIAS"],
      "android.injected.signing.key.password"   => ENV["KEY_PASSWORD"]
    }
  )
end

# 方式二：使用 sign_apk action
lane :sign_apk do
  sign_apk(
    apk_path:         "app/build/outputs/apk/release/app-release-unsigned.apk",
    signed_apk_path:  "app/build/outputs/apk/release/app-release.apk",
    keystore_path:    ENV["KEYSTORE_FILE"],
    alias:            ENV["KEY_ALIAS"],
    storepass:        ENV["STORE_PASSWORD"],
    keypass:          ENV["KEY_PASSWORD"]
  )
end
```

### 上传 Google Play（supply / upload_to_play_store）

```ruby
# fastlane/Supplyfile
package_name("com.example.app")
track("internal")
json_key(ENV["GOOGLE_PLAY_JSON_KEY"])    # 服务账号 JSON 文件路径

# Fastfile 中使用
lane :deploy_play_store do |options|
  track = options[:track] || "internal"  # internal | alpha | beta | production

  upload_to_play_store(
    track:                    track,
    aab:                      "app/build/outputs/bundle/release/app-release.aab",
    json_key:                 ENV["GOOGLE_PLAY_JSON_KEY"],
    release_status:           "draft",   # draft | completed | halted | inProgress
    rollout:                  "0.1",     # 10% 灰度（production track 用）
    skip_upload_apk:          true,
    skip_upload_metadata:     false,
    skip_upload_screenshots:  true,
    version_codes_to_retain:  [last_version_code, last_version_code - 1]
  )
end

# 仅晋升 track（不重新上传）
lane :promote_to_production do
  upload_to_play_store(
    track:            "internal",
    track_promote_to: "production",
    rollout:          "1.0"
  )
end
```

```shell
# 获取 Google Play JSON Key：
# Google Play Console → Setup → API access → Create service account
# 在 Google Cloud Console 下载 JSON Key
# 在 Google Play Console 赋予该账号"发布管理员"权限
```

### 上传 Firebase App Distribution

```ruby
# 需要插件：fastlane-plugin-firebase_app_distribution
lane :distribute_firebase do
  firebase_app_distribution(
    app:                        ENV["FIREBASE_APP_ID"],
    firebase_cli_token:         ENV["FIREBASE_TOKEN"],    # firebase login:ci 生成
    # 或使用服务账号
    service_credentials_file:   "google-service-account.json",

    ipa_path:                   "./build/MyApp.ipa",      # iOS
    # apk_path:                 "./app/build/outputs/apk/release/app-release.apk",  # Android

    groups:                     "qa-team, beta-testers",
    release_notes:              "Build #{ENV['CI_BUILD_NUMBER']}: #{last_git_commit[:message]}",
    release_notes_file:         "RELEASE_NOTES.txt"       # 或从文件读取
  )
end
```

### Android 自动截图（screengrab）

```ruby
# fastlane/Screengrabfile
locales(["zh-CN", "en-US"])
clear_previous_screenshots(true)
app_apk_path("app/build/outputs/apk/debug/app-debug.apk")
tests_apk_path("app/build/outputs/apk/androidTest/debug/app-debug-androidTest.apk")
output_directory("fastlane/screenshots")

# 运行
lane :screenshots do
  capture_android_screenshots
end
```

---

## 常用通用 Actions

### Git 操作

```ruby
# 确保 Git 干净
ensure_git_status_clean

# 拉取最新代码
git_pull

# 提交版本号变更
git_commit(
  path:    ["./MyApp/Info.plist", "./MyApp.xcodeproj"],
  message: "Bump version to #{get_version_number} (#{get_build_number})"
)

# 打 Tag
add_git_tag(
  tag:     "v#{get_version_number}-#{get_build_number}",
  message: "Release #{get_version_number}"
)

# 推送 commit 和 tag
push_to_git_remote(
  remote:      "origin",
  local_branch: "main",
  tags:         true
)

# 获取最近一次 commit 信息
commit = last_git_commit
puts commit[:message]        # commit 信息
puts commit[:author]         # 作者
puts commit[:commit_hash]    # SHA
```

### 通知

```ruby
# Slack 通知
slack(
  message:     "#{lane_context[SharedValues::LANE_NAME]} 构建成功 🎉",
  success:     true,
  slack_url:   ENV["SLACK_WEBHOOK_URL"],
  payload: {
    "版本"    => get_version_number,
    "Build"   => get_build_number,
    "分支"    => git_branch,
    "提交"    => last_git_commit[:message]
  },
  default_payloads: [:git_branch, :git_author]
)

# 钉钉通知（需插件 fastlane-plugin-dingtalk）
dingtalk(
  webhook:  ENV["DINGTALK_WEBHOOK"],
  message:  "iOS 打包成功，版本 #{get_version_number}"
)
```

### 工具类

```ruby
# 读取 JSON 文件
data = load_json(json_path: "config.json")
puts data["version"]

# 在 CI 环境中检测
if is_ci
  puts "running on CI"
end

# 获取当前 Git 分支
branch = git_branch

# 确保当前在正确分支
ensure_git_branch(branch: "main")

# 清理构建目录
clean_build_artifacts

# 执行 Shell 命令
sh("echo Hello World")
sh("pod install --repo-update")

# 获取命令输出
output = sh("git rev-parse --short HEAD").strip
puts "当前 commit: #{output}"
```

---

## 环境与配置管理

### 多环境配置

```
fastlane/
├── .env              # 通用配置（所有环境）
├── .env.default      # 默认配置
├── .env.staging      # Staging 环境
└── .env.production   # Production 环境
```

```shell
# fastlane/.env
SLACK_WEBHOOK_URL=https://hooks.slack.com/xxx
MATCH_GIT_URL=git@github.com:org/certificates.git

# fastlane/.env.staging
APP_IDENTIFIER=com.example.app.staging
APP_STORE_CONNECT_TEAM_ID=XXXXXXXXXX
MATCH_TYPE=adhoc

# fastlane/.env.production
APP_IDENTIFIER=com.example.app
MATCH_TYPE=appstore
```

```shell
# 指定环境运行
fastlane beta --env staging
fastlane release --env production
```

```ruby
# Fastfile 中引用
lane :build do
  match(
    type:           ENV["MATCH_TYPE"],
    app_identifier: ENV["APP_IDENTIFIER"]
  )
end
```

### Appfile

```ruby
# fastlane/Appfile

# iOS
app_identifier ENV["APP_IDENTIFIER"] || "com.example.app"
apple_id        ENV["APPLE_ID"]      || "apple@example.com"
team_id         ENV["TEAM_ID"]                               # Developer Team ID
itc_team_id     ENV["ITC_TEAM_ID"]                           # App Store Connect Team ID

# Android
json_key_file   ENV["GOOGLE_PLAY_JSON_KEY"]
package_name    ENV["PACKAGE_NAME"] || "com.example.app"
```

---

## 插件系统

### 安装与管理插件

```shell
# 搜索插件
fastlane search_plugins firebase

# 安装插件（自动更新 Pluginfile 和 Gemfile）
fastlane add_plugin firebase_app_distribution
fastlane add_plugin versioning
fastlane add_plugin pgyer              # 蒲公英分发
fastlane add_plugin fir_cli            # fir.im 分发

# 安装项目已有插件
fastlane install_plugins

# 查看已安装插件
cat fastlane/Pluginfile
```

### 常用插件推荐

| 插件 | 用途 |
|------|------|
| `firebase_app_distribution` | Firebase 内测分发 |
| `versioning` | Android 版本号管理 |
| `pgyer` | 蒲公英内测分发 |
| `fir_cli` | fir.im 内测分发 |
| `appicon` | 自动生成各尺寸应用图标 |
| `increment_build_number_in_plist` | plist 版本管理 |
| `sonar` | SonarQube 代码扫描 |
| `slack_upload` | 上传文件到 Slack |
| `badge` | 在应用图标上添加 Badge（版本号/环境标识）|
| `changelog` | 自动生成 changelog |
| `jira` | 更新 Jira 票据状态 |

---

## 与 CI/CD 平台集成

### GitHub Actions + Fastlane（iOS）

```yaml
# .github/workflows/ios-release.yml
name: iOS Release

on:
  push:
    tags: ['v*.*.*']

jobs:
  release:
    runs-on: macos-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true        # 自动 bundle install 并缓存

      - name: Install CocoaPods
        run: bundle exec pod install --repo-update

      - name: Setup SSH for Match
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.MATCH_SSH_PRIVATE_KEY }}

      - name: Run Fastlane
        run: bundle exec fastlane release
        env:
          MATCH_PASSWORD:          ${{ secrets.MATCH_PASSWORD }}
          ASC_KEY_ID:              ${{ secrets.ASC_KEY_ID }}
          ASC_ISSUER_ID:           ${{ secrets.ASC_ISSUER_ID }}
          ASC_KEY_CONTENT:         ${{ secrets.ASC_KEY_CONTENT }}
          SLACK_WEBHOOK_URL:       ${{ secrets.SLACK_WEBHOOK_URL }}
          APP_IDENTIFIER:          com.example.app
          TEAM_ID:                 ${{ secrets.APPLE_TEAM_ID }}
```

### GitHub Actions + Fastlane（Android）

```yaml
# .github/workflows/android-release.yml
name: Android Release

on:
  push:
    tags: ['v*.*.*']

jobs:
  release:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.2'
          bundler-cache: true

      - name: Decode Keystore
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d > app/release.jks

      - name: Run Fastlane
        run: bundle exec fastlane android deploy_play_store
        env:
          KEYSTORE_FILE:         app/release.jks
          STORE_PASSWORD:        ${{ secrets.STORE_PASSWORD }}
          KEY_ALIAS:             ${{ secrets.KEY_ALIAS }}
          KEY_PASSWORD:          ${{ secrets.KEY_PASSWORD }}
          GOOGLE_PLAY_JSON_KEY:  ${{ secrets.GOOGLE_PLAY_JSON_KEY }}
          FIREBASE_APP_ID:       ${{ secrets.FIREBASE_APP_ID }}
          FIREBASE_TOKEN:        ${{ secrets.FIREBASE_TOKEN }}
```

### GitLab CI + Fastlane（iOS）

```yaml
# .gitlab-ci.yml
stages: [build, distribute]

variables:
  LC_ALL: "en_US.UTF-8"
  LANG:   "en_US.UTF-8"

.ios-base:
  tags: [macos, xcode]              # 需要 macOS Self-hosted Runner
  before_script:
    - bundle install --path vendor/bundle
    - bundle exec pod install

ios-beta:
  extends: .ios-base
  stage: distribute
  script:
    - bundle exec fastlane beta
  environment:
    name: testflight
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

ios-release:
  extends: .ios-base
  stage: distribute
  script:
    - bundle exec fastlane release
  environment:
    name: app-store
  when: manual
  rules:
    - if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/'
```

---

## 完整示例

### iOS 完整 Fastfile

```ruby
# fastlane/Fastfile
fastlane_version "2.200.0"
default_platform(:ios)

APP_ID      = ENV["APP_IDENTIFIER"] || "com.example.app"
OUTPUT_DIR  = "./build"

platform :ios do
  before_all do
    setup_ci if is_ci                  # CI 环境下自动配置 Keychain
  end

  # ── 私有辅助 Lane ─────────────────────────────
  private_lane :prepare_signing do |opts|
    match(
      type:           opts[:type] || "adhoc",
      app_identifier: APP_ID,
      readonly:       is_ci
    )
  end

  private_lane :build do |opts|
    prepare_signing(type: opts[:match_type])
    increment_build_number(
      build_number: is_ci ? ENV["CI_PIPELINE_IID"] : Time.now.strftime("%Y%m%d%H%M")
    )
    build_app(
      scheme:           "MyApp",
      workspace:        "MyApp.xcworkspace",
      configuration:    opts[:configuration] || "Release",
      export_method:    opts[:export_method] || "ad-hoc",
      output_directory: OUTPUT_DIR,
      output_name:      "MyApp.ipa",
      clean:            true
    )
  end

  private_lane :notify do |opts|
    slack(
      message:   opts[:message],
      success:   opts.fetch(:success, true),
      slack_url: ENV["SLACK_WEBHOOK_URL"],
      payload: {
        "版本"  => "#{get_version_number} (#{get_build_number})",
        "分支"  => git_branch,
        "提交"  => last_git_commit[:message]
      }
    ) if ENV["SLACK_WEBHOOK_URL"]
  end

  # ── 测试 ───────────────────────────────────────
  desc "运行单元测试和 UI 测试"
  lane :test do
    run_tests(
      workspace:        "MyApp.xcworkspace",
      scheme:           "MyAppTests",
      devices:          ["iPhone 15 Pro"],
      code_coverage:    true,
      output_directory: "fastlane/test_output",
      output_types:     "html,junit"
    )
  end

  # ── 内测分发（Firebase） ────────────────────────
  desc "打包并上传到 Firebase App Distribution"
  lane :firebase do
    build(match_type: "adhoc", export_method: "ad-hoc")

    firebase_app_distribution(
      app:               ENV["FIREBASE_APP_ID"],
      firebase_cli_token: ENV["FIREBASE_TOKEN"],
      ipa_path:          "#{OUTPUT_DIR}/MyApp.ipa",
      groups:            "qa-team",
      release_notes:     "[#{git_branch}] #{last_git_commit[:message]}"
    )

    notify(message: "iOS 内测包已上传到 Firebase 🚀")
  end

  # ── TestFlight ─────────────────────────────────
  desc "打包并上传到 TestFlight"
  lane :beta do
    api_key = app_store_connect_api_key(
      key_id:      ENV["ASC_KEY_ID"],
      issuer_id:   ENV["ASC_ISSUER_ID"],
      key_content: ENV["ASC_KEY_CONTENT"]
    )

    build(match_type: "appstore", export_method: "app-store")

    upload_to_testflight(
      api_key:                           api_key,
      ipa:                               "#{OUTPUT_DIR}/MyApp.ipa",
      changelog:                         last_git_commit[:message],
      skip_waiting_for_build_processing: true
    )

    notify(message: "iOS TestFlight 包已上传 ✅")
  end

  # ── App Store 发布 ─────────────────────────────
  desc "构建并提交到 App Store 审核"
  lane :release do
    ensure_git_branch(branch: "main")
    ensure_git_status_clean

    api_key = app_store_connect_api_key(
      key_id:      ENV["ASC_KEY_ID"],
      issuer_id:   ENV["ASC_ISSUER_ID"],
      key_content: ENV["ASC_KEY_CONTENT"]
    )

    build(match_type: "appstore", export_method: "app-store")

    version = get_version_number
    upload_to_app_store(
      api_key:           api_key,
      ipa:               "#{OUTPUT_DIR}/MyApp.ipa",
      app_version:       version,
      submit_for_review: true,
      automatic_release: false,
      force:             true,
      skip_screenshots:  true
    )

    add_git_tag(tag: "v#{version}-#{get_build_number}")
    push_to_git_remote(tags: true)

    notify(message: "iOS v#{version} 已提交 App Store 审核 🎉")
  end

  error do |lane, exception|
    notify(message: "Lane [#{lane}] 失败：#{exception.message}", success: false)
  end
end
```

### Android 完整 Fastfile

```ruby
# fastlane/Fastfile (Android)
fastlane_version "2.200.0"
default_platform(:android)

platform :android do
  # ── 测试 ───────────────────────────────────────
  desc "运行单元测试"
  lane :test do
    gradle(task: "test")
  end

  # ── 构建 ───────────────────────────────────────
  desc "构建 Release AAB"
  lane :build do |opts|
    version_code = (ENV["CI_PIPELINE_IID"] || Time.now.strftime("%Y%m%d%H%M")).to_i

    gradle(
      task:       "bundle",
      build_type: "Release",
      properties: {
        "versionCode"                                         => version_code,
        "android.injected.signing.store.file"                => ENV["KEYSTORE_FILE"],
        "android.injected.signing.store.password"            => ENV["STORE_PASSWORD"],
        "android.injected.signing.key.alias"                 => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password"              => ENV["KEY_PASSWORD"]
      }
    )
  end

  # ── Firebase 内测分发 ──────────────────────────
  desc "打包并上传到 Firebase App Distribution"
  lane :firebase do
    gradle(task: "assemble", build_type: "Release",
      properties: {
        "android.injected.signing.store.file"     => ENV["KEYSTORE_FILE"],
        "android.injected.signing.store.password" => ENV["STORE_PASSWORD"],
        "android.injected.signing.key.alias"      => ENV["KEY_ALIAS"],
        "android.injected.signing.key.password"   => ENV["KEY_PASSWORD"]
      })

    firebase_app_distribution(
      app:               ENV["FIREBASE_APP_ID"],
      firebase_cli_token: ENV["FIREBASE_TOKEN"],
      apk_path:          lane_context[SharedValues::GRADLE_APK_OUTPUT_PATH],
      groups:            "qa-team",
      release_notes:     last_git_commit[:message]
    )
  end

  # ── Google Play 发布 ───────────────────────────
  desc "上传到 Google Play"
  lane :deploy do |opts|
    track = opts[:track] || "internal"

    build

    upload_to_play_store(
      track:                   track,
      aab:                     lane_context[SharedValues::GRADLE_AAB_OUTPUT_PATH],
      json_key:                ENV["GOOGLE_PLAY_JSON_KEY"],
      skip_upload_apk:         true,
      skip_upload_metadata:    true,
      skip_upload_screenshots: true
    )

    slack(
      message:   "Android 已上传到 Google Play [#{track}] 🚀",
      slack_url: ENV["SLACK_WEBHOOK_URL"]
    ) if ENV["SLACK_WEBHOOK_URL"]
  end

  error do |lane, exception|
    slack(
      message:   "Android Lane [#{lane}] 失败：#{exception.message}",
      success:   false,
      slack_url: ENV["SLACK_WEBHOOK_URL"]
    ) if ENV["SLACK_WEBHOOK_URL"]
  end
end
```

---

## 常用命令

```shell
# 列出所有可用 lane
fastlane lanes
fastlane list

# 运行指定 lane
fastlane ios beta
fastlane android deploy track:internal

# 运行并传参
fastlane deploy env:production version:2.0.0

# 在 CI 环境运行（跳过交互，输出更简洁）
fastlane ios release --env production

# 指定日志级别
fastlane beta --verbose

# 查看某个 action 的文档
fastlane action gym
fastlane action upload_to_testflight

# 更新 Fastlane
bundle update fastlane              # Bundler 方式
```

---

## 最佳实践

1. **用 Bundler 锁定版本**：项目中始终使用 `Gemfile` + `Gemfile.lock` 固定 Fastlane 和插件版本，避免升级导致的兼容性问题，CI 上用 `bundle exec fastlane` 执行。

2. **敏感信息环境变量化**：证书密码、API Key、Webhook URL 一律通过环境变量注入，`.env` 文件加入 `.gitignore`，CI 中使用平台提供的 Secret 管理。

3. **使用 App Store Connect API Key**：替代 Apple ID + 2FA 登录，CI 更稳定，不受双因素认证干扰。Key 内容 Base64 编码后存为 Secret。

4. **match 统一管理签名**：团队共用同一套证书，CI 上使用 `readonly: true` 只拉取不生成，避免证书冲突。

5. **lane 职责单一**：每条 lane 只做一件事，通过 `private_lane` 封装公共逻辑，`lane` 之间组合调用，提高可读性和复用性。

6. **CI 上设置 `setup_ci`**：在 `before_all` 中调用 `setup_ci if is_ci`，它会自动创建临时 Keychain，防止签名时弹出密码对话框。

7. **构建编号与 CI 流水线号绑定**：用 CI 的 Pipeline/Build Number 作为 Build Number，保证每次构建唯一且可追溯。

8. **提取公共配置到 Appfile**：Bundle ID、Team ID、Apple ID 等写入 Appfile，lane 中无需重复声明。

9. **善用 `error` 钩子**：在 `error` 回调中统一处理失败通知，避免在每条 lane 中写 `begin/rescue`。

10. **本地测试与 CI 保持一致**：开发者本地运行 `bundle exec fastlane test` 应与 CI 结果完全一致，通过 `.env` 文件模拟 CI 变量调试。
