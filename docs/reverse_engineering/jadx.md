# JADX

JADX 是一款将 Android APK、AAR、DEX、Class 文件反编译为可读 Java/Kotlin 代码的开源工具，提供命令行（jadx）和图形界面（jadx-gui）两种使用方式。

---

## 核心概念

| 概念           | 说明                                                 |
| -------------- | ---------------------------------------------------- |
| **DEX → Java** | 将 Dalvik 字节码还原为 Java 源码                     |
| **Smali**      | DEX 的人类可读汇编格式，jadx 可同时显示              |
| **资源反编译** | 还原 `res/`、`AndroidManifest.xml`、`resources.arsc` |
| **混淆还原**   | 自动识别常见混淆模式并重命名                         |
| **jadx-gui**   | 内置搜索、交叉引用、反混淆编辑的图形界面             |
| **jadx CLI**   | 批量导出、脚本集成、CI 流水线使用                    |

---

## 安装

### 方式一：Homebrew（macOS）

```bash
brew install jadx
```

### 方式二：下载预编译包

```bash
# 从 GitHub Releases 下载最新版
# https://github.com/skylot/jadx/releases
# 下载 jadx-<version>.zip，解压后加入 PATH
unzip jadx-1.5.0.zip -d ~/tools/jadx
export PATH="$PATH:~/tools/jadx/bin"
```

### 方式三：从源码构建

```bash
git clone https://github.com/skylot/jadx.git
cd jadx
./gradlew dist
# 产物在 build/jadx/bin/
```

### 验证安装

```bash
jadx --version
# jadx version: 1.5.0
```

---

## 快速入门

### 反编译 APK 到目录

```bash
# 将 APK 反编译到 ./output 目录
jadx -d output app.apk

# 指定线程数加速（默认 4）
jadx -d output --threads-count 8 app.apk
```

### 启动图形界面

```bash
jadx-gui app.apk
# 或直接打开 jadx-gui 再拖入文件
```

### 支持的输入格式

| 格式     | 说明                 |
| -------- | -------------------- |
| `.apk`   | Android 应用包       |
| `.aab`   | Android App Bundle   |
| `.aar`   | Android 库文件       |
| `.dex`   | Dalvik 可执行文件    |
| `.class` | Java 字节码          |
| `.jar`   | Java 归档包          |
| `.zip`   | 包含以上格式的压缩包 |

---

## jadx-gui 使用

### 界面布局

```
┌─────────────────────────────────────────────────────┐
│  菜单栏  工具栏                                        │
├──────────────┬──────────────────────────────────────┤
│              │                                       │
│  包结构树     │   代码查看区                           │
│  (左侧面板)   │   (右侧主区域，支持多标签)               │
│              │                                       │
├──────────────┴──────────────────────────────────────┤
│  日志 / 搜索结果面板（底部）                            │
└─────────────────────────────────────────────────────┘
```

### 常用快捷键

| 快捷键            | 功能                        |
| ----------------- | --------------------------- |
| `Ctrl+N`          | 全局类名搜索                |
| `Ctrl+F`          | 当前文件内搜索              |
| `Ctrl+Shift+F`    | 全文本搜索（跨文件）        |
| `Ctrl+Click`      | 跳转到定义                  |
| `Alt+←` / `Alt+→` | 后退 / 前进                 |
| `X`               | 查看所有引用（Find Usages） |
| `Ctrl+G`          | 跳转到行号                  |
| `Ctrl+W`          | 关闭当前标签                |
| `F5`              | 刷新/重新反编译             |
| `Ctrl+S`          | 保存当前文件                |
| `Ctrl+Shift+S`    | 全部保存                    |
| `Ctrl+T`          | 在新标签中打开              |
| `Ctrl+E`          | 切换最近文件                |

### 查看 Smali 代码

右键点击类 → **Show Smali** 或在代码区底部切换 **Java / Smali** 标签页，可对照查看 Dalvik 字节码。

---

## CLI 命令参考

### 基础语法

```bash
jadx [options] <input file>
jadx-gui [options] <input file>
```

### 常用选项

```bash
# 输出目录
-d, --output-dir <dir>

# 仅导出 Java 源码（不含资源）
--output-dir-src <dir>

# 仅导出资源
--output-dir-res <dir>

# 导出为单个 jar
--export-gradle

# 显示所有警告
--show-bad-code

# 不混淆重命名（保留 a/b/c 等短名）
--no-imports

# 反混淆（自动重命名）
--deobf

# 反混淆最小长度阈值（默认 3）
--deobf-min 3

# 反混淆最大长度阈值（默认 64）
--deobf-max 64

# 将反混淆映射写入文件
--deobf-map-file mapping.txt

# 从文件读取已有映射
--deobf-use-sourcename

# 线程数
--threads-count 8

# 跳过资源反编译（只反编译代码）
--no-res

# 跳过源码反编译（只还原资源）
--no-src

# 输出 jadx 日志
-v, --verbose

# 静默模式
-q, --quiet
```

### 常用组合命令

```bash
# 仅反编译源码，跳过资源（速度更快）
jadx --no-res -d output app.apk

# 开启反混淆，8 线程
jadx --deobf --threads-count 8 -d output app.apk

# 仅查看某个 dex 文件
jadx -d output classes2.dex

# 从 aab 导出 Java 源码
jadx -d output app.aab

# 导出 Gradle 项目结构（便于在 Android Studio 中打开）
jadx --export-gradle -d output app.apk

# 生成反混淆映射文件
jadx --deobf --deobf-map-file mapping.txt -d output app.apk
```

---

## 反混淆功能

### 自动反混淆

启用后，jadx 会将 `a`、`b`、`c` 等短名重命名为带有语义的名称（如 `class1`、`field2`），使代码更易读。

```bash
jadx --deobf -d output app.apk
```

### 手动重命名（jadx-gui）

在 GUI 中右键点击类、方法或字段 → **Rename** → 输入新名称 → 映射关系自动保存到 `.jadx` 配置目录，下次加载同一文件时自动应用。

### 使用 ProGuard/R8 映射文件

```bash
# 应用已有的 mapping.txt 还原混淆名
jadx --deobf-map-file mapping.txt -d output app.apk
```

### 从源码还原（SourceFile 属性）

部分 APK 保留了 `SourceFile` 调试属性，jadx 可自动利用这些信息还原类名：

```bash
jadx --deobf-use-sourcename -d output app.apk
```

---

## 搜索功能

### 全局类搜索（Ctrl+N）

输入类名关键词，支持驼峰缩写：

- 输入 `MA` → 匹配 `MainActivity`
- 输入 `http` → 匹配所有含 http 的类名

### 全文搜索（Ctrl+Shift+F）

搜索所有反编译代码中的字符串、方法调用、字段引用。

**搜索选项：**

| 选项         | 说明             |
| ------------ | ---------------- |
| **Class**    | 仅在类名中搜索   |
| **Method**   | 仅在方法名中搜索 |
| **Field**    | 仅在字段名中搜索 |
| **Code**     | 在代码文本中搜索 |
| **Comment**  | 在注释中搜索     |
| **Resource** | 在资源文件中搜索 |

### 查找引用（X 键 / Find Usages）

选中类、方法或字段后按 `X`，列出所有调用/引用位置，是分析代码流的核心功能。

---

## 实战场景

### 场景一：分析网络请求

**目标**：找到 App 调用的 API 地址。

```
1. Ctrl+Shift+F → 搜索 "https://" 或 "http://"
2. 过滤 Code 类型，找到硬编码的 URL
3. 搜索 "OkHttpClient" 或 "Retrofit" 定位网络层
4. 右键 → Find Usages 追踪调用链
```

**常见位置：**

```java
// strings.xml 中的 base_url
<string name="base_url">https://api.example.com/</string>

// BuildConfig 常量
public static final String API_URL = "https://api.example.com/";

// Retrofit 注解
@GET("/v1/users/{id}")
Call<User> getUser(@Path("id") String id);
```

### 场景二：分析加密逻辑

**目标**：找到 App 使用的加密算法和密钥。

```bash
# 搜索关键词
Cipher          # javax.crypto.Cipher
SecretKeySpec   # AES/DES 密钥
MessageDigest   # MD5/SHA 哈希
Mac             # HMAC
Base64          # 编码
```

典型代码：

```java
// AES 加密示例（反编译后）
SecretKeySpec keySpec = new SecretKeySpec("hardcoded_key_16".getBytes(), "AES");
Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
cipher.init(Cipher.ENCRYPT_MODE, keySpec, new IvParameterSpec(iv));
byte[] encrypted = cipher.doFinal(plaintext.getBytes());
```

### 场景三：绕过签名校验

**目标**：找到签名验证逻辑并定位关键判断。

```
1. 搜索 "getPackageInfo" 或 "signatures"
2. 搜索 "signature" 或 "sign"
3. 搜索 "PackageManager"
4. 找到返回 boolean 的校验方法 → 结合 Frida hook 绕过
```

```java
// 常见签名校验模式
public boolean verifySignature(Context ctx) {
    PackageInfo info = ctx.getPackageManager()
        .getPackageInfo(ctx.getPackageName(), PackageManager.GET_SIGNATURES);
    String sig = toHex(info.signatures[0].toByteArray());
    return EXPECTED_SIG.equals(sig);
}
```

### 场景四：分析 Native 层调用

**目标**：找到 JNI 方法及对应的 `.so` 文件。

```
1. 搜索 "native" 关键词 → 找到 native 方法声明
2. 搜索 "System.loadLibrary" → 找到加载的 .so 名称
3. 搜索 "JNI_OnLoad" 或 "RegisterNatives"
4. 配合 IDA Pro / Ghidra 分析对应 so 文件
```

```java
// native 方法声明（反编译后）
public class NativeHelper {
    static {
        System.loadLibrary("native-lib");  // 对应 libnative-lib.so
    }
    public native String encrypt(String input);
    public native boolean verify(byte[] data);
}
```

### 场景五：提取硬编码密钥/Token

```
搜索关键词列表：
- "secret"
- "password" / "passwd"
- "token"
- "apikey" / "api_key"
- "authorization"
- "Bearer"
- "private_key"
- "-----BEGIN"   # PEM 格式证书/密钥
```

### 场景六：分析 WebView 与 JS Bridge

**目标**：找到 JS 注入接口。

```java
// 搜索 addJavascriptInterface
webView.addJavascriptInterface(new JsBridge(this), "NativeApp");

// 搜索 @JavascriptInterface 注解
@JavascriptInterface
public String getUserToken() {
    return sharedPreferences.getString("token", "");
}
```

---

## 导出与集成

### 导出为 Gradle 项目

```bash
jadx --export-gradle -d output app.apk
```

导出后结构：

```
output/
├── app/
│   ├── build.gradle
│   └── src/main/
│       ├── java/          # 反编译的 Java 源码
│       ├── res/           # 资源文件
│       └── AndroidManifest.xml
└── build.gradle
```

可直接用 Android Studio 打开进行二次开发或深度分析。

### 与 Frida 联动

jadx 定位代码位置 → Frida 动态 hook：

```python
# jadx 找到方法签名后，Frida 编写 hook
import frida

script_code = """
Java.perform(function() {
    // jadx 中找到：com.example.app.util.CryptoHelper.encrypt(String)
    var CryptoHelper = Java.use("com.example.app.util.CryptoHelper");
    CryptoHelper.encrypt.implementation = function(input) {
        console.log("[*] encrypt called with: " + input);
        var result = this.encrypt(input);
        console.log("[*] encrypt result: " + result);
        return result;
    };
});
"""

device = frida.get_usb_device()
session = device.attach("com.example.app")
script = session.create_script(script_code)
script.load()
```

### 与 apktool 联动

| 工具        | 适合场景                        |
| ----------- | ------------------------------- |
| **jadx**    | 查看 Java/Kotlin 逻辑，搜索代码 |
| **apktool** | 修改 smali、替换资源、重打包    |

```bash
# jadx 分析逻辑，找到目标 smali 位置
jadx-gui app.apk

# apktool 解包修改
apktool d app.apk -o app_decoded

# 修改 smali 后重打包
apktool b app_decoded -o app_patched.apk

# 签名
apksigner sign --ks debug.jks app_patched.apk
```

---

## 高级用法

### 分析多 DEX 包

现代 APK 通常包含 `classes.dex`、`classes2.dex`、`classes3.dex`，jadx 自动合并处理：

```bash
# APK 中所有 dex 自动加载
jadx -d output app.apk

# 单独分析某个 dex
jadx -d output classes2.dex
```

### 分析 Flutter APK

Flutter APK 的业务逻辑在 `libapp.so`（Dart AOT 编译产物），Java 层较少。可以：

1. jadx 查看 Java 层的 Flutter Engine 初始化代码
2. 使用 `reFlutter` 工具重打包 Flutter APK 以拦截流量
3. 使用 `blutter` 分析 Dart AOT 产物中的符号

```bash
# 搜索 Flutter 相关类
io.flutter.embedding.android.FlutterActivity
io.flutter.plugin.common.MethodChannel
```

### 分析 React Native APK

业务逻辑打包在 `assets/index.android.bundle`（JS bundle），可直接提取分析：

```bash
# 用 jadx 打开 APK 后在资源区找到 bundle
# 或用 unzip 直接提取
unzip app.apk assets/index.android.bundle -d rn_extracted

# 格式化 bundle（通常是压缩的 JS）
node -e "
const fs = require('fs');
const code = fs.readFileSync('index.android.bundle', 'utf8');
fs.writeFileSync('bundle.js', code);
"

# 或使用 hermes-dec 反编译 Hermes 字节码
npm install -g hermes-dec
hermes-dec assets/index.android.bundle
```

### 分析加固 APK

常见加固厂商（360、腾讯乐固、梆梆、爱加密）会将真实 DEX 加密，jadx 直接打开只能看到壳代码。

**处理流程：**

```
1. jadx 打开 → 只有壳代码 (com.stub.*, com.shield.*)
2. 运行 App → 壳在内存中解密真实 DEX
3. Frida dump 内存中的 DEX：
```

```javascript
// Frida 脚本：dump 内存中的 DEX
Java.perform(function () {
  var runtime = Java.use("java.lang.Runtime");
  var vmDebug = Java.use("dalvik.system.VMDebug");

  // 方法一：枚举加载的 ClassLoader
  Java.enumerateClassLoaders({
    onMatch: function (loader) {
      console.log("ClassLoader: " + loader);
    },
    onComplete: function () {},
  });
});

// 方法二：内存扫描 DEX magic bytes
// DEX magic: 64 65 78 0a 30 33 35 00 (dex\n035\0)
var ranges = Process.enumerateRanges("r--");
ranges.forEach(function (range) {
  var pattern = "64 65 78 0a 30 33 35 00";
  Memory.scan(range.base, range.size, pattern, {
    onMatch: function (address, size) {
      console.log("DEX found at: " + address);
      // 读取 DEX 文件大小（偏移 0x20 处的 uint32）
      var dexSize = address.add(0x20).readU32();
      var dexBytes = address.readByteArray(dexSize);
      // 保存到文件
      var f = new File("/data/local/tmp/dump_" + address + ".dex", "wb");
      f.write(dexBytes);
      f.close();
    },
    onError: function (reason) {},
    onComplete: function () {},
  });
});
```

```bash
# dump 完成后用 jadx 分析
adb pull /data/local/tmp/dump_*.dex .
jadx -d output dump_*.dex
```

### 脚本化批量分析

jadx 支持通过 Gradle API 或直接调用 CLI 实现自动化：

```bash
#!/bin/bash
# 批量反编译目录下所有 APK
for apk in *.apk; do
    name="${apk%.apk}"
    echo "Processing: $apk"
    jadx --no-res --deobf --threads-count 8 -d "output/$name" "$apk"
done
```

```python
# Python 脚本：调用 jadx 并搜索关键字
import subprocess
import os

def decompile_and_search(apk_path, keywords):
    output_dir = "/tmp/jadx_output"

    # 反编译
    subprocess.run([
        "jadx", "--no-res", "--deobf", "-d", output_dir, apk_path
    ], check=True)

    results = {}
    for keyword in keywords:
        found = []
        for root, dirs, files in os.walk(output_dir):
            for f in files:
                if f.endswith(".java"):
                    filepath = os.path.join(root, f)
                    with open(filepath, "r", errors="ignore") as fh:
                        content = fh.read()
                        if keyword in content:
                            found.append(filepath)
        results[keyword] = found

    return results

# 使用示例
results = decompile_and_search(
    "target.apk",
    ["https://", "SecretKeySpec", "getSignatures", "addJavascriptInterface"]
)
for kw, files in results.items():
    print(f"\n[{kw}] found in {len(files)} files:")
    for f in files:
        print(f"  {f}")
```

---

## JADX API（Java 库）

jadx 也可作为 Java 库嵌入到工具中：

```kotlin
// build.gradle
dependencies {
    implementation("io.github.skylot:jadx-core:1.5.0")
}
```

```java
import jadx.api.JadxArgs;
import jadx.api.JadxDecompiler;
import jadx.api.JavaClass;

public class JadxExample {
    public static void main(String[] args) throws Exception {
        JadxArgs jadxArgs = new JadxArgs();
        jadxArgs.setInputFile(new File("app.apk"));
        jadxArgs.setOutputDir(new File("output"));
        jadxArgs.setDeobfuscationOn(true);
        jadxArgs.setThreadsCount(4);

        try (JadxDecompiler jadx = new JadxDecompiler(jadxArgs)) {
            jadx.load();

            // 遍历所有类
            for (JavaClass cls : jadx.getClasses()) {
                System.out.println("Class: " + cls.getFullName());
                System.out.println(cls.getCode());
            }

            // 搜索特定类
            JavaClass targetClass = jadx.searchJavaClassByFullName(
                "com.example.app.MainActivity"
            );
            if (targetClass != null) {
                System.out.println(targetClass.getCode());
            }

            // 导出全部
            jadx.saveSources();
        }
    }
}
```

---

## 常见问题

### 反编译结果有大量 `/* JADX WARNING */`

这是 jadx 遇到无法完美还原的字节码时添加的警告注释，常见原因：

| 警告类型                           | 原因                   | 处理方式            |
| ---------------------------------- | ---------------------- | ------------------- |
| `unknown type`                     | 混淆后类型丢失         | 结合 Smali 视图分析 |
| `code is unreachable`              | 死代码优化             | 忽略                |
| `declared exception is not thrown` | try-catch 结构异常     | 查看 Smali 原始逻辑 |
| `inconsistent code`                | 字节码与 Java 语义不符 | 切换查看 Smali      |

```bash
# 显示所有有问题的代码块
jadx --show-bad-code -d output app.apk
```

### 类找不到（ClassNotFound）

```
可能原因：
1. 多 DEX：类在其他 dex 文件中 → 直接分析 APK（自动合并）
2. 动态加载：类在运行时从网络/本地加载 → 需要 Frida 在运行时 dump
3. 加固：类被加密 → 脱壳后再分析
```

### 反编译速度慢

```bash
# 增加线程数
jadx --threads-count 16 -d output app.apk

# 跳过资源反编译
jadx --no-res -d output app.apk

# 增大 JVM 内存
export JAVA_OPTS="-Xmx4g"
jadx -d output app.apk
```

### GUI 打开 APK 崩溃

```bash
# 命令行查看错误原因
jadx -d /tmp/test app.apk

# 常见原因：
# 1. APK 损坏 → 尝试 zip -T app.apk 验证
# 2. Java 版本过低 → jadx 需要 Java 11+
# 3. 内存不足 → 增大堆内存
java -version  # 确认 Java 11+
```

### 反混淆后名称冲突

```bash
# 使用唯一数字后缀避免冲突（默认行为）
# 或禁用反混淆使用原始短名
jadx -d output app.apk  # 不加 --deobf
```

---

## jadx vs 其他工具

| 工具                | 输入        | 输出             | 特点                      |
| ------------------- | ----------- | ---------------- | ------------------------- |
| **jadx**            | APK/DEX/JAR | Java/Kotlin 源码 | 最易读，带 GUI，反混淆强  |
| **apktool**         | APK         | Smali + 资源     | 支持重打包，修改 APK 必用 |
| **enjarify**        | APK/DEX     | JAR              | 配合 procyon 使用         |
| **CFR**             | JAR/Class   | Java 源码        | Java 反编译质量高         |
| **Procyon**         | JAR/Class   | Java 源码        | 处理复杂语法更好          |
| **Ghidra**          | Native .so  | C 伪码           | NSA 开源，分析 native 层  |
| **IDA Pro**         | Native .so  | C 伪码           | 商业工具，业界标准        |
| **ByteCode Viewer** | APK/JAR     | 多种反编译器     | 集成多个后端，可对比      |

**推荐组合：**

```
静态分析 APK Java 层  →  jadx-gui
修改/重打包 APK       →  apktool + apksigner
分析 Native .so       →  Ghidra 或 IDA Pro
动态调试/Hook         →  Frida
一键分析（快速入手）  →  MobSF（集成框架）
```

---

## 完整分析流程示例

以分析一个未知 APK 的加密通信为例：

```bash
# 1. 基础信息收集
jadx --no-res -d output target.apk
ls output/sources/  # 查看包名结构

# 2. 启动 GUI 深入分析
jadx-gui target.apk
```

```
# 3. GUI 中的分析步骤

Step 1: 查看 AndroidManifest.xml
  → 确认包名、权限、入口 Activity、Service、Receiver

Step 2: Ctrl+Shift+F 搜索网络相关
  → "OkHttpClient" "Retrofit" "HttpURLConnection" "volley"
  → 找到网络层实现

Step 3: 追踪 API 请求构建
  → 找到 BaseUrl 和请求方法
  → 查看请求头（Authorization、Token）

Step 4: 搜索加密逻辑
  → "Cipher" "SecretKeySpec" "Mac" "sign"
  → 找到请求签名算法

Step 5: 查看密钥来源
  → 是硬编码？从服务器获取？从 native 层获取？
  → 如果是 native 层 → 提取对应 .so 文件 → Ghidra 分析

Step 6: 编写 Frida hook 验证
  → 动态 hook 加密方法，打印明文和密文
  → 验证静态分析结论
```

```python
# 4. Frida 验证脚本（基于 jadx 分析结果）
import frida, sys

PACKAGE = "com.target.app"

script_code = """
Java.perform(function() {
    // 基于 jadx 找到的类名和方法名
    var RequestSigner = Java.use("com.target.app.network.RequestSigner");

    RequestSigner.sign.overload("java.lang.String", "java.lang.String").implementation = function(method, body) {
        console.log("\\n[RequestSigner.sign]");
        console.log("  method: " + method);
        console.log("  body: " + body);
        var result = this.sign(method, body);
        console.log("  signature: " + result);
        return result;
    };

    var CryptoUtil = Java.use("com.target.app.util.CryptoUtil");
    CryptoUtil.encrypt.implementation = function(plaintext) {
        console.log("\\n[CryptoUtil.encrypt]");
        console.log("  plaintext: " + plaintext);
        var result = this.encrypt(plaintext);
        console.log("  ciphertext: " + result);
        return result;
    };
});
"""

device = frida.get_usb_device()
session = device.attach(PACKAGE)
script = session.create_script(script_code)
script.on("message", lambda msg, data: print(msg))
script.load()
sys.stdin.read()
```

---
