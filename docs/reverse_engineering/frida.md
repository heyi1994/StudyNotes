# Frida

Frida 是一款跨平台的动态插桩框架，支持 Android、iOS、Windows、macOS、Linux。它将 JavaScript 引擎（V8/Duktape）注入目标进程，让你无需重编译即可在运行时 hook 函数、追踪调用、修改内存、拦截网络——是移动端逆向分析和安全研究的标准工具。

**核心概念：**

| 概念 | 说明 |
|------|------|
| **frida-server** | 运行在目标设备上的守护进程，接收 PC 端指令并执行注入 |
| **frida-agent** | 被注入进目标进程的 JS 运行时 |
| **Script** | 用户编写的 JavaScript 钩子脚本 |
| **Interceptor** | 拦截 Native（C/C++）函数的核心 API |
| **Java API** | 操作 Android Java 层的 API（`Java.use`、`Java.perform`）|
| **ObjC API** | 操作 iOS Objective-C 层的 API |
| **Memory** | 内存读写、扫描、分配 API |
| **Module** | 目标进程加载的共享库（.so/.dylib/.dll）|
| **Stalker** | 代码追踪引擎，可跟踪指令级执行流 |
| **RPC** | 脚本与 PC 端 Python 双向通信机制 |

---

## 安装与环境准备

### 安装 Frida 工具链（PC 端）

```shell
# 安装 frida 核心库和命令行工具
pip install frida frida-tools

# 验证安装
frida --version
frida-ps --version

# 升级
pip install --upgrade frida frida-tools
```

### Android 环境配置

**步骤 1：下载 frida-server**

```shell
# 查看当前 frida 版本
frida --version   # 例如：16.2.1

# 在 https://github.com/frida/frida/releases 下载对应版本的 frida-server
# 根据设备架构选择：
#   arm64  → frida-server-16.2.1-android-arm64.xz
#   arm    → frida-server-16.2.1-android-arm.xz
#   x86_64 → frida-server-16.2.1-android-x86_64.xz（模拟器常用）
#   x86    → frida-server-16.2.1-android-x86.xz

# 查看设备架构
adb shell getprop ro.product.cpu.abi
```

**步骤 2：推送并启动 frida-server**

```shell
# 解压并推送到设备
xz -d frida-server-16.2.1-android-arm64.xz
adb push frida-server-16.2.1-android-arm64 /data/local/tmp/frida-server
adb shell chmod 755 /data/local/tmp/frida-server

# 启动（需要 Root）
adb shell su -c "/data/local/tmp/frida-server &"

# 验证：列出设备进程
frida-ps -U          # -U = USB 设备
frida-ps -U -a       # 只显示正在运行的应用
frida-ps -U -i       # 只显示已安装的应用

# 端口转发（可选，用于网络连接）
adb forward tcp:27042 tcp:27042
frida-ps -H 127.0.0.1:27042
```

**步骤 3：无 Root 注入（frida-gadget）**

```shell
# 将 frida-gadget.so 注入 APK，适用于无 Root 设备
pip install objection

# 使用 objection patchapk 自动注入
objection patchapk -s target.apk

# 安装重打包后的 APK
adb install -r target.objection.apk
```

### iOS 环境配置

```shell
# iOS 需要越狱设备
# 在 Cydia / Sileo 中安装 Frida（BigBoss 或官方源）
# 源地址：https://build.frida.re

# 验证
frida-ps -U

# 也可通过 SSH 手动安装
dpkg -i frida_16.x.x_iphoneos-arm.deb
```

---

## 命令行工具

### frida（交互式 REPL）

```shell
# 附加到正在运行的进程（包名或进程名）
frida -U com.example.app
frida -U -n "Target App"

# 附加并执行脚本
frida -U com.example.app -l hook.js

# 启动应用并注入（-f = spawn）
frida -U -f com.example.app --no-pause

# 重启应用并等待（调试初始化阶段）
frida -U -f com.example.app   # 暂停在入口，需手动 %resume

# 连接到远程主机
frida -H 192.168.1.100:27042 com.example.app
```

### frida-ps（进程列表）

```shell
frida-ps -U             # 所有进程
frida-ps -U -a          # 运行中的应用
frida-ps -U -i          # 已安装的应用（含包名）
frida-ps -U | grep wechat
```

### frida-trace（自动追踪函数调用）

```shell
# 追踪所有 open 系统调用
frida-trace -U com.example.app -i "open"

# 追踪 libc 中的函数
frida-trace -U com.example.app -i "recv*" -i "send*"

# 追踪 Java 方法（Android）
frida-trace -U com.example.app -j "com.example.app.LoginActivity.checkPassword"

# 追踪某个 so 中的所有导出函数
frida-trace -U com.example.app -I "libssl.so"

# 追踪 ObjC 方法（iOS）
frida-trace -U com.example.app -m "-[LoginViewController verify*]"

# 结果会在 __handlers__/ 目录生成可编辑的 JS 处理函数
```

### frida-dump / objection

```shell
# objection：基于 Frida 的一体化逆向工具
pip install objection

# 启动并注入
objection -g com.example.app explore

# objection 常用命令（在 explore shell 中）
android hooking list classes              # 列出所有已加载的类
android hooking list class_methods com.example.app.LoginActivity
android hooking watch class com.example.app.LoginActivity
android hooking watch method com.example.app.LoginActivity.checkPassword --dump-args --dump-return
android sslpinning disable               # 一键禁用 SSL Pinning
android root disable                     # 绕过 Root 检测
android intent launch_activity com.example.app.MainActivity
memory list modules                      # 列出已加载模块
memory dump all mem.dmp                  # dump 内存
ios hooking list classes
ios sslpinning disable
```

---

## JavaScript API 核心

### Java 层 Hook（Android）

所有操作 Java 对象的代码必须在 `Java.perform()` 回调中执行，确保在 Java VM 初始化后运行。

```javascript
Java.perform(function () {

    // ── 基础 Hook ─────────────────────────────────
    var LoginActivity = Java.use("com.example.app.LoginActivity");

    // Hook 实例方法
    LoginActivity.checkPassword.implementation = function (password) {
        console.log("[*] checkPassword called, password: " + password);

        // 调用原方法
        var result = this.checkPassword(password);
        console.log("[*] checkPassword returned: " + result);

        // 修改返回值
        return true;
    };

    // ── 方法重载（Overload）────────────────────────
    // 当方法有多个重载时，必须用 .overload() 指定参数类型
    LoginActivity.checkPassword.overload("java.lang.String").implementation = function (pwd) {
        return true;
    };

    LoginActivity.checkPassword.overload("java.lang.String", "boolean").implementation = function (pwd, remember) {
        console.log("pwd=" + pwd + ", remember=" + remember);
        return true;
    };

    // ── 修改参数 ──────────────────────────────────
    var CryptoHelper = Java.use("com.example.app.CryptoHelper");
    CryptoHelper.encrypt.implementation = function (data, key) {
        console.log("[*] encrypt data: " + data);
        // 替换 key 参数
        var result = this.encrypt(data, "my_custom_key");
        console.log("[*] encrypt result: " + result);
        return result;
    };

    // ── 访问 / 修改字段 ────────────────────────────
    var UserModel = Java.use("com.example.app.UserModel");
    UserModel.isVip.value = true;                        // 静态字段

    // 实例字段需要先获取实例
    Java.choose("com.example.app.UserModel", {
        onMatch: function (instance) {
            console.log("找到实例，isVip = " + instance.isVip.value);
            instance.isVip.value = true;                 // 修改实例字段
        },
        onComplete: function () {
            console.log("枚举完成");
        }
    });

    // ── 调用静态方法 ──────────────────────────────
    var Utils = Java.use("com.example.app.Utils");
    var result = Utils.staticMethod("arg1", 42);

    // ── 创建对象实例 ──────────────────────────────
    var StringBuilder = Java.use("java.lang.StringBuilder");
    var sb = StringBuilder.$new("Hello");
    sb.append(", World");
    console.log(sb.toString());

    // ── 调用私有方法（通过反射）────────────────────
    var clazz = Java.use("com.example.app.SecretClass");
    var instance = clazz.$new();
    // 私有方法可以直接 hook，无需特殊处理
    clazz.privateMethod.implementation = function () {
        return "modified";
    };

});
```

### 处理特殊类型

```javascript
Java.perform(function () {

    // ── byte[] 数组 ───────────────────────────────
    var Cipher = Java.use("javax.crypto.Cipher");
    Cipher.doFinal.overload("[B").implementation = function (input) {
        // 将 Java byte[] 转为十六进制字符串
        var hexInput = Array.from(input, b => ('0' + (b & 0xFF).toString(16)).slice(-2)).join('');
        console.log("[*] Cipher.doFinal input (hex): " + hexInput);

        var result = this.doFinal(input);

        var hexResult = Array.from(result, b => ('0' + (b & 0xFF).toString(16)).slice(-2)).join('');
        console.log("[*] Cipher.doFinal output (hex): " + hexResult);
        return result;
    };

    // ── 字符串 <-> byte[] 转换 ─────────────────────
    function bytesToHex(bytes) {
        return Array.from(bytes, b => ('0' + (b & 0xFF).toString(16)).slice(-2)).join('');
    }
    function bytesToString(bytes) {
        return Java.use("java.lang.String").$new(bytes, "UTF-8");
    }
    function stringToBytes(str) {
        return Java.use("java.lang.String").$new(str).getBytes("UTF-8");
    }

    // ── List / Map 操作 ───────────────────────────
    var ArrayList = Java.use("java.util.ArrayList");
    var list = ArrayList.$new();
    list.add("item1");
    list.add("item2");
    console.log("size: " + list.size());
    console.log("item: " + list.get(0));

    // 遍历 Java 集合
    var iterator = list.iterator();
    while (iterator.hasNext()) {
        console.log(iterator.next());
    }

    // ── 枚举 (Enum) ────────────────────────────────
    var Status = Java.use("com.example.app.Status");
    console.log(Status.ACTIVE.value);           // 枚举值
    // 设置枚举字段
    instance.status.value = Status.ACTIVE.value;

    // ── 接口 / 匿名类 ──────────────────────────────
    var Runnable = Java.use("java.lang.Runnable");
    var myRunnable = Java.implement(Runnable, {
        run: function () {
            console.log("Runnable.run called");
        }
    });

    // ── 内部类 ────────────────────────────────────
    var InnerClass = Java.use("com.example.app.Outer$Inner");

    // ── 类加载器问题（热修复/插件化框架）────────────
    Java.enumerateClassLoaders({
        onMatch: function (loader) {
            try {
                var clazz = loader.loadClass("com.example.app.HiddenClass");
                // 找到目标类后使用该 classLoader
                Java.classFactory.loader = loader;
                var HiddenClass = Java.use("com.example.app.HiddenClass");
                console.log("找到 HiddenClass");
            } catch (e) {}
        },
        onComplete: function () {}
    });

});
```

### Native 层 Hook（Interceptor）

```javascript
// ── 基础 Native Hook ─────────────────────────────
// 通过导出名 hook（需要函数未被 strip）
Interceptor.attach(Module.findExportByName("libc.so", "open"), {
    onEnter: function (args) {
        // args 是 NativePointer 数组，对应函数参数
        var path = args[0].readUtf8String();
        console.log("[open] path: " + path);

        // 保存参数供 onLeave 使用
        this.path = path;
    },
    onLeave: function (retval) {
        // retval 是返回值（NativePointer）
        var fd = retval.toInt32();
        console.log("[open] " + this.path + " -> fd=" + fd);

        // 修改返回值
        if (this.path.includes("xposed")) {
            retval.replace(-1);  // 返回 -1（失败）
        }
    }
});

// ── 通过地址 hook ─────────────────────────────────
var baseAddr = Module.findBaseAddress("libtarget.so");
var targetOffset = 0x1234;  // IDA/Ghidra 中找到的偏移
var targetAddr = baseAddr.add(targetOffset);

Interceptor.attach(targetAddr, {
    onEnter: function (args) {
        console.log("called at offset 0x" + targetOffset.toString(16));
        console.log("arg0: " + args[0]);
        console.log("arg1: " + args[1].readUtf8String());
    },
    onLeave: function (retval) {
        console.log("return: " + retval);
    }
});

// ── 完全替换函数（NativeCallback）────────────────
var original = Module.findExportByName("libtarget.so", "isRooted");
var replacement = new NativeCallback(function () {
    console.log("[*] isRooted hooked, returning false");
    return 0;
}, "int", []);  // 返回类型, [参数类型...]
Interceptor.replace(original, replacement);

// ── 调用 Native 函数（NativeFunction）────────────
var strlen = new NativeFunction(
    Module.findExportByName("libc.so", "strlen"),
    "size_t",   // 返回类型
    ["pointer"] // 参数类型
);
var len = strlen(Memory.allocUtf8String("hello"));
console.log("strlen: " + len);  // 5
```

### 内存操作（Memory）

```javascript
// ── 读内存 ────────────────────────────────────────
var ptr = ptr("0x12345678");
ptr.readS8()        // 有符号 8 位整数
ptr.readU8()        // 无符号 8 位整数
ptr.readS16()
ptr.readU16()
ptr.readS32()
ptr.readU32()
ptr.readS64()
ptr.readU64()
ptr.readFloat()
ptr.readDouble()
ptr.readPointer()   // 读指针（平台指针大小）
ptr.readByteArray(16)               // 读 16 字节，返回 ArrayBuffer
ptr.readUtf8String()                // 读 UTF-8 字符串
ptr.readUtf8String(32)              // 最多读 32 字节
ptr.readUtf16String()
ptr.readCString()

// ── 写内存 ────────────────────────────────────────
ptr.writeS32(42)
ptr.writeU8(0xFF)
ptr.writePointer(ptr("0xdeadbeef"))
ptr.writeByteArray([0x90, 0x90, 0x90])  // 写 NOP 指令
ptr.writeUtf8String("modified")

// ── 分配内存 ──────────────────────────────────────
var buf = Memory.alloc(64)                    // 分配 64 字节
var strPtr = Memory.allocUtf8String("hello") // 分配字符串

// ── 内存扫描 ──────────────────────────────────────
Memory.scan(
    ptr("0x40000000"),   // 起始地址
    0x10000,             // 扫描长度
    "48 89 ?? ?? 90",    // 特征码（? 是通配符）
    {
        onMatch: function (address, size) {
            console.log("找到匹配: " + address);
        },
        onComplete: function () {
            console.log("扫描完成");
        }
    }
);

// ── 修改内存保护属性 ──────────────────────────────
Memory.protect(ptr("0x40001000"), 0x1000, "rwx");

// ── 模块信息 ──────────────────────────────────────
Process.enumerateModules().forEach(function (m) {
    console.log(m.name + " @ " + m.base + " size=" + m.size);
});

var libssl = Process.findModuleByName("libssl.so");
console.log("libssl base: " + libssl.base);

// 枚举模块导出函数
Module.enumerateExports("libtarget.so").forEach(function (exp) {
    console.log(exp.type + " " + exp.name + " @ " + exp.address);
});

// 枚举模块导入函数
Module.enumerateImports("libtarget.so").forEach(function (imp) {
    console.log(imp.module + "!" + imp.name + " -> " + imp.address);
});

// 查找导出函数地址
var openPtr = Module.findExportByName(null, "open");  // null = 所有模块
```

### 进程与线程

```javascript
// 枚举线程
Process.enumerateThreads().forEach(function (thread) {
    console.log("tid=" + thread.id + " state=" + thread.state);
});

// 枚举堆内存范围
Process.enumerateMallocs().forEach(function (chunk) {
    console.log("malloc: " + chunk.address + " size=" + chunk.size);
});

// 获取进程信息
console.log("pid: " + Process.id);
console.log("arch: " + Process.arch);       // arm64 / x64 / ia32 / arm
console.log("platform: " + Process.platform); // android / linux / darwin / windows
console.log("pageSize: " + Process.pageSize);

// 获取当前线程的调用栈
var backtrace = Thread.backtrace(this.context, Backtracer.ACCURATE);
backtrace.forEach(function (addr) {
    var sym = DebugSymbol.fromAddress(addr);
    console.log("  " + addr + " " + sym);
});

// 栈回溯辅助函数
function printBacktrace(context) {
    Thread.backtrace(context, Backtracer.ACCURATE)
        .map(DebugSymbol.fromAddress)
        .forEach(s => console.log("  " + s));
}
```

---

## Python 控制端（frida Python API）

Python 端负责启动/附加进程、加载脚本、与脚本通信。

### 基础模板

```python
import frida
import sys

# ── 附加模式 ─────────────────────────────────────
def on_message(message, data):
    if message["type"] == "send":
        print("[JS->Python]", message["payload"])
    elif message["type"] == "error":
        print("[Error]", message["stack"])

# 连接 USB 设备
device = frida.get_usb_device()

# 附加到已运行的进程
session = device.attach("com.example.app")

with open("hook.js", "r") as f:
    script_code = f.read()

script = session.create_script(script_code)
script.on("message", on_message)
script.load()

# 阻塞直到手动停止
sys.stdin.read()
```

### Spawn 模式（应用启动时注入）

```python
import frida
import sys

def on_message(message, data):
    if message["type"] == "send":
        print("[*]", message["payload"])
    elif message["type"] == "error":
        print("[!]", message["stack"])

device = frida.get_usb_device()

# spawn：启动应用并暂停在入口
pid = device.spawn(["com.example.app"])
session = device.attach(pid)

script = session.create_script(open("hook.js").read())
script.on("message", on_message)
script.load()

# resume：恢复应用执行
device.resume(pid)

sys.stdin.read()
```

### RPC 双向通信

```javascript
// hook.js — 导出可从 Python 调用的函数
rpc.exports = {
    // Python 可以直接调用这些函数
    getEncryptionKey: function () {
        var key = Java.use("com.example.app.CryptoManager").KEY.value;
        return key;
    },
    setFlag: function (value) {
        Java.perform(function () {
            Java.use("com.example.app.Settings").debugMode.value = value;
        });
    },
    callMethod: function (input) {
        var result;
        Java.perform(function () {
            result = Java.use("com.example.app.Utils").process(input);
        });
        return result;
    }
};

// JS 主动向 Python 发送消息
send({ type: "key_found", key: "abc123" });
send("simple string message");

// 发送二进制数据
send({ type: "binary" }, bytes);   // bytes 是 ArrayBuffer
```

```python
# main.py
import frida

device = frida.get_usb_device()
pid = device.spawn(["com.example.app"])
session = device.attach(pid)

script = session.create_script(open("hook.js").read())

def on_message(message, data):
    if message["type"] == "send":
        payload = message["payload"]
        if isinstance(payload, dict) and payload.get("type") == "key_found":
            print("发现密钥：", payload["key"])
        if data:
            print("二进制数据：", data.hex())

script.on("message", on_message)
script.load()
device.resume(pid)

# 调用 JS 导出的 RPC 函数
key = script.exports.get_encryption_key()     # 驼峰自动转下划线
print("密钥：", key)

script.exports.set_flag(True)
result = script.exports.call_method("input_data")
print("结果：", result)
```

---

## 常用 Hook 场景

### SSL Pinning 绕过

```javascript
// 方法一：Hook OkHttp3 CertificatePinner
Java.perform(function () {
    var CertificatePinner = Java.use("okhttp3.CertificatePinner");

    CertificatePinner.check.overload("java.lang.String", "java.util.List")
        .implementation = function (hostname, peerCertificates) {
            console.log("[*] Bypassed CertificatePinner.check for: " + hostname);
            // 什么都不做即为通过
        };

    // check$okhttp 是某些版本的变体
    try {
        CertificatePinner["check$okhttp"].implementation = function () {};
    } catch (e) {}
});

// 方法二：Hook TrustManager（系统级）
Java.perform(function () {
    var X509TrustManager = Java.use("javax.net.ssl.X509TrustManager");
    var SSLContext = Java.use("javax.net.ssl.SSLContext");

    var TrustManager = Java.registerClass({
        name: "com.frida.TrustManager",
        implements: [X509TrustManager],
        methods: {
            checkClientTrusted: function (chain, authType) {},
            checkServerTrusted: function (chain, authType) {},
            getAcceptedIssuers: function () { return []; }
        }
    });

    var TrustManagers = [TrustManager.$new()];
    var ctx = SSLContext.getInstance("TLS");
    ctx.init(null, TrustManagers, null);

    var SSLSocketFactory = Java.use("javax.net.ssl.HttpsURLConnection");
    SSLSocketFactory.setDefaultSSLSocketFactory(ctx.getSocketFactory());

    console.log("[*] TrustManager 已替换");
});

// 方法三：objection 一键绕过
// android sslpinning disable
```

### Root 检测绕过

```javascript
Java.perform(function () {
    // 1. 绕过 RootBeer / 常见 Root 检测库
    try {
        var RootBeer = Java.use("com.scottyab.rootbeer.RootBeer");
        RootBeer.isRooted.implementation = function () { return false; };
        RootBeer.isRootedWithBusyBoxCheck.implementation = function () { return false; };
    } catch (e) {}

    // 2. 绕过文件检测（检测 /system/bin/su 等）
    var File = Java.use("java.io.File");
    File.exists.implementation = function () {
        var name = this.getName();
        if (name === "su" || name === "magisk" || name === "busybox") {
            console.log("[*] 拦截文件检测: " + this.getAbsolutePath());
            return false;
        }
        return this.exists();
    };

    // 3. 绕过 Runtime.exec 执行 su
    var Runtime = Java.use("java.lang.Runtime");
    Runtime.exec.overload("java.lang.String").implementation = function (cmd) {
        if (cmd.indexOf("su") !== -1 || cmd.indexOf("which") !== -1) {
            console.log("[*] 拦截命令执行: " + cmd);
            throw Java.use("java.io.IOException").$new("Permission denied");
        }
        return this.exec(cmd);
    };

    // 4. 绕过系统属性检测
    var SystemProperties = Java.use("android.os.SystemProperties");
    SystemProperties.get.overload("java.lang.String").implementation = function (key) {
        if (key === "ro.debuggable" || key === "service.adb.root") {
            return "0";
        }
        return this.get(key);
    };
});

// Native 层绕过（hook fopen 拦截读取 /proc/self/maps）
Interceptor.attach(Module.findExportByName("libc.so", "fopen"), {
    onEnter: function (args) {
        var path = args[0].readUtf8String();
        if (path && (path.includes("magisk") || path.includes("xposed"))) {
            console.log("[*] 拦截 fopen: " + path);
            args[0] = Memory.allocUtf8String("/dev/null");
        }
    }
});
```

### 加密算法追踪

```javascript
Java.perform(function () {

    // ── AES 加解密 ────────────────────────────────
    var Cipher = Java.use("javax.crypto.Cipher");

    Cipher.getInstance.overload("java.lang.String").implementation = function (algo) {
        console.log("[Cipher.getInstance] algorithm: " + algo);
        return this.getInstance(algo);
    };

    Cipher.init.overload("int", "java.security.Key").implementation = function (opmode, key) {
        var modeStr = opmode === 1 ? "ENCRYPT" : opmode === 2 ? "DECRYPT" : "WRAP/UNWRAP";
        console.log("[Cipher.init] mode: " + modeStr);

        // 提取密钥字节
        var keyBytes = key.getEncoded();
        var keyHex = Array.from(keyBytes, b => ('0' + (b & 0xFF).toString(16)).slice(-2)).join('');
        console.log("[Cipher.init] key (hex): " + keyHex);

        return this.init(opmode, key);
    };

    Cipher.doFinal.overload("[B").implementation = function (input) {
        var inputHex = Array.from(input, b => ('0' + (b & 0xFF).toString(16)).slice(-2)).join('');
        console.log("[Cipher.doFinal] input (hex): " + inputHex);

        var result = this.doFinal(input);

        var resultHex = Array.from(result, b => ('0' + (b & 0xFF).toString(16)).slice(-2)).join('');
        console.log("[Cipher.doFinal] output (hex): " + resultHex);
        return result;
    };

    // ── MessageDigest (MD5/SHA) ────────────────────
    var MessageDigest = Java.use("java.security.MessageDigest");
    MessageDigest.getInstance.overload("java.lang.String").implementation = function (algo) {
        console.log("[MessageDigest] algorithm: " + algo);
        return this.getInstance(algo);
    };

    MessageDigest.digest.overload("[B").implementation = function (input) {
        var inputStr = Java.use("java.lang.String").$new(input, "UTF-8");
        console.log("[MessageDigest.digest] input: " + inputStr);
        var result = this.digest(input);
        var hex = Array.from(result, b => ('0' + (b & 0xFF).toString(16)).slice(-2)).join('');
        console.log("[MessageDigest.digest] result: " + hex);
        return result;
    };

    // ── Mac（HMAC）────────────────────────────────
    var Mac = Java.use("javax.crypto.Mac");
    Mac.doFinal.overload("[B").implementation = function (input) {
        var inputHex = Array.from(input, b => ('0' + (b & 0xFF).toString(16)).slice(-2)).join('');
        console.log("[Mac.doFinal] input: " + inputHex);
        var result = this.doFinal(input);
        var resultHex = Array.from(result, b => ('0' + (b & 0xFF).toString(16)).slice(-2)).join('');
        console.log("[Mac.doFinal] HMAC: " + resultHex);
        return result;
    };
});
```

### 网络请求拦截（OkHttp）

```javascript
Java.perform(function () {

    // ── 拦截 OkHttp3 请求与响应 ───────────────────
    var OkHttpClient = Java.use("okhttp3.OkHttpClient");
    var Request      = Java.use("okhttp3.Request");
    var Response     = Java.use("okhttp3.Response");
    var Buffer       = Java.use("okio.Buffer");

    // Hook RealCall.execute
    var RealCall = Java.use("okhttp3.internal.connection.RealCall");
    RealCall.execute.implementation = function () {
        var request = this.request();
        console.log("\n[OkHttp] ===== 请求 =====");
        console.log("URL: " + request.url().toString());
        console.log("Method: " + request.method());

        // 打印请求头
        var headers = request.headers();
        for (var i = 0; i < headers.size(); i++) {
            console.log("Header: " + headers.name(i) + ": " + headers.value(i));
        }

        // 打印请求体
        var body = request.body();
        if (body !== null) {
            var buffer = Buffer.$new();
            body.writeTo(buffer);
            console.log("Body: " + buffer.readUtf8());
        }

        var response = this.execute();

        console.log("[OkHttp] ===== 响应 =====");
        console.log("Code: " + response.code());

        // 读取响应体（需要 peek，否则会消费 body）
        var peekBody = response.peekBody(1024 * 1024);  // 最多 1MB
        console.log("Body: " + peekBody.string());

        return response;
    };

    // ── Retrofit + OkHttp 组合拦截 ────────────────
    // Hook OkHttp Interceptor（更优雅的方式）
    var Interceptor = Java.use("okhttp3.Interceptor");
    var Chain = Java.use("okhttp3.Interceptor$Chain");

    // 注册自定义拦截器需要在 OkHttpClient 构建时注入
    // 更直接的方式是 hook 底层 Exchange 类
    var Exchange = Java.use("okhttp3.internal.connection.Exchange");
    Exchange.writeRequestHeaders.implementation = function (request) {
        console.log("[Exchange] URL: " + request.url() + " Method: " + request.method());
        return this.writeRequestHeaders(request);
    };
});
```

### 打印调用栈

```javascript
// 通用调用栈追踪工具函数
function stackTrace() {
    var Exception = Java.use("java.lang.Exception");
    var exception = Exception.$new("FridaStackTrace");
    var stackElements = exception.getStackTrace();
    var stack = "\n[调用栈]\n";
    for (var i = 0; i < Math.min(stackElements.length, 20); i++) {
        stack += "  " + stackElements[i].toString() + "\n";
    }
    return stack;
}

// 使用
Java.perform(function () {
    var Target = Java.use("com.example.app.SensitiveClass");
    Target.sensitiveMethod.implementation = function () {
        console.log("[*] sensitiveMethod called" + stackTrace());
        return this.sensitiveMethod();
    };
});
```

---

## Stalker（代码追踪）

Stalker 是 Frida 的代码追踪引擎，可以在指令级别追踪代码执行流。

```javascript
// 追踪目标线程的所有执行指令
var targetTid = Process.enumerateThreads()[0].id;

Stalker.follow(targetTid, {
    events: {
        call: true,     // 追踪 call 指令
        ret:  false,    // 追踪 ret 指令
        exec: false,    // 追踪所有指令（开销极大）
        block: false,   // 追踪基本块
        compile: false  // 追踪 JIT 编译
    },

    onReceive: function (events) {
        // events 是 ArrayBuffer，包含追踪到的事件
        var reader = Stalker.parse(events);
        for (var event = reader.next(); event !== null; event = reader.next()) {
            if (event[0] === "call") {
                console.log("call: " + event[1] + " -> " + event[2]);
            }
        }
    },

    // 转换器：在 JIT 编译时修改代码
    transform: function (iterator) {
        var instruction;
        while ((instruction = iterator.next()) !== null) {
            // 在每条 call 指令前插入日志
            if (instruction.mnemonic === "call") {
                iterator.putCallout(function (context) {
                    // context 包含寄存器状态
                    console.log("call to: 0x" + context.pc.toString(16));
                });
            }
            iterator.keep();
        }
    }
});

// 停止追踪
setTimeout(function () {
    Stalker.unfollow(targetTid);
    Stalker.flush();
}, 5000);
```

---

## frida-server 持久化

```shell
# 方法一：开机自启（Magisk 模块方式，在 /data/adb/service.d/ 放启动脚本）
cat > /data/adb/service.d/start-frida.sh << 'EOF'
#!/system/bin/sh
while [ "$(getprop sys.boot_completed)" != "1" ]; do
    sleep 2
done
sleep 5
/data/local/tmp/frida-server &
EOF
chmod 755 /data/adb/service.d/start-frida.sh

# 方法二：通过 adb shell 后台运行
adb shell "su -c 'nohup /data/local/tmp/frida-server > /dev/null 2>&1 &'"

# 确认是否启动
adb shell "ps -A | grep frida"
```

---

## 反检测

部分应用会检测 Frida 的存在：

```javascript
// 检测点：
// 1. /proc/self/maps 中含 frida 相关路径
// 2. 端口 27042 被监听
// 3. frida-agent 的 so 文件特征
// 4. 线程名含 "gmain"、"gdbus" 等 GLib 特征

// 绕过：hook fopen / open 拦截对 /proc/self/maps 的读取
Interceptor.attach(Module.findExportByName("libc.so", "fopen"), {
    onEnter: function (args) {
        var path = args[0].readUtf8String();
        if (path === "/proc/self/maps") {
            // 替换为空文件
            args[0] = Memory.allocUtf8String("/dev/null");
        }
    }
});

// 绕过端口检测：hook connect/bind
Interceptor.attach(Module.findExportByName("libc.so", "connect"), {
    onEnter: function (args) {
        // 解析 sockaddr，如果是 27042 端口则返回失败
    }
});

// 使用 --gadget 模式（注入 gadget.so 到 APK）可以绕过大部分运行时检测
// 因为没有 frida-server，检测端口 / 进程名等方式失效
```

---

## 完整实战脚本示例

### 抓取 HTTPS 明文数据

```python
# intercept_https.py
import frida, sys, json

HOOK_SCRIPT = """
Java.perform(function () {
    // Hook OkHttp3 RequestBody
    var RequestBody = Java.use("okhttp3.RequestBody");
    var MediaType = Java.use("okhttp3.MediaType");

    // Hook Retrofit 接口调用前的参数
    // Hook 关键业务方法
    var ApiService = Java.use("com.example.app.network.ApiServiceImpl");
    ApiService.login.implementation = function (username, password) {
        send({
            type: "login_attempt",
            username: username,
            password: password
        });
        return this.login(username, password);
    };

    // Hook Response 解析
    var GsonBuilder = Java.use("com.google.gson.GsonBuilder");
    var Gson = Java.use("com.google.gson.Gson");
    Gson.fromJson.overload("java.lang.String", "java.lang.Class").implementation = function(json, clazz) {
        console.log("[Gson.fromJson] JSON: " + json);
        return this.fromJson(json, clazz);
    };
});
"""

def on_message(message, data):
    if message["type"] == "send":
        payload = message["payload"]
        print(f"[{payload['type']}] user={payload.get('username')} pwd={payload.get('password')}")

device = frida.get_usb_device()
pid = device.spawn(["com.example.app"])
session = device.attach(pid)
script = session.create_script(HOOK_SCRIPT)
script.on("message", on_message)
script.load()
device.resume(pid)
print("[*] 开始监听，按 Enter 退出")
sys.stdin.readline()
```

### 动态分析 Native so 函数

```javascript
// native_analysis.js
(function () {

    var targetLib = "libtarget.so";
    var module = Process.findModuleByName(targetLib);

    if (!module) {
        console.log("[-] 未找到 " + targetLib);
        return;
    }

    console.log("[+] " + targetLib + " @ " + module.base + " size=0x" + module.size.toString(16));

    // 枚举并 hook 所有导出函数
    Module.enumerateExports(targetLib).forEach(function (exp) {
        if (exp.type !== "function") return;

        Interceptor.attach(exp.address, {
            onEnter: function (args) {
                var stack = Thread.backtrace(this.context, Backtracer.ACCURATE)
                    .map(DebugSymbol.fromAddress).join('\n\t');
                send({
                    type: "native_call",
                    name: exp.name,
                    address: exp.address.toString(),
                    backtrace: stack
                });
            }
        });
        console.log("[+] Hooked export: " + exp.name);
    });

    // 专项 hook：hook JNI_OnLoad 获取动态注册的 Native 函数
    var JNI_OnLoad = Module.findExportByName(targetLib, "JNI_OnLoad");
    if (JNI_OnLoad) {
        Interceptor.attach(JNI_OnLoad, {
            onLeave: function (retval) {
                console.log("[*] JNI_OnLoad returned");
                // 此时动态注册函数已完成，可以枚举
            }
        });
    }

    // Hook RegisterNatives 捕获动态注册
    var RegisterNatives = Module.findExportByName("libart.so", "_ZN3art3JNI15RegisterNativesEP7_JNIEnvP7_jclassPK15JNINativeMethodi");
    if (RegisterNatives) {
        Interceptor.attach(RegisterNatives, {
            onEnter: function (args) {
                var count = args[3].toInt32();
                for (var i = 0; i < count; i++) {
                    var offset = i * Process.pointerSize * 3;
                    var name = args[2].add(offset).readPointer().readCString();
                    var sig  = args[2].add(offset + Process.pointerSize).readPointer().readCString();
                    var func = args[2].add(offset + Process.pointerSize * 2).readPointer();
                    console.log("[RegisterNatives] " + name + " " + sig + " @ " + func);
                }
            }
        });
    }
})();
```

---

## 常用工具整合

| 工具 | 用途 |
|------|------|
| `objection` | 基于 Frida 的一体化逆向 shell，内置常用 hook |
| `frida-ios-dump` | 砸壳 iOS 加密 IPA |
| `r2frida` | Radare2 + Frida 集成，静动态分析结合 |
| `Brida` | Burp Suite + Frida 联动，修改加密请求 |
| `fridump` | 内存转储工具 |
| `dexdump` / `jadx` | 搭配 Frida 反编译定位 hook 点 |

---

## 最佳实践

1. **版本严格对齐**：frida（PC）、frida-server（设备）、frida-tools 必须版本完全一致，否则会出现协议不兼容错误。

2. **`Java.perform` 包裹所有 Java 操作**：直接在外层调用 `Java.use` 会因 VM 未初始化而报错，所有 Java 操作必须放入 `Java.perform(function() { ... })` 中。

3. **方法重载必须用 `.overload()`**：遇到 `Error: ambiguous overload` 时，必须通过 `.overload("java.lang.String", ...)` 精确指定参数类型。

4. **先 hook 再 resume**：Spawn 模式下要在 `device.resume()` 之前加载并激活脚本，否则会错过初始化阶段的关键逻辑。

5. **Native hook 用 try-catch 包裹**：`Module.findExportByName` 可能返回 null（函数被 strip 或名称混淆），调用前先判断是否为 null。

6. **大量日志影响性能**：在 `Interceptor.attach` 的 `onEnter/onLeave` 中避免过多 `console.log`，高频调用的函数应使用 `send()` 批量发送或先记录再汇总。

7. **使用 `Process.arch` 做架构适配**：32 位和 64 位的指针大小、函数调用约定不同，用 `Process.pointerSize` 代替硬编码的 4 或 8。

8. **利用 `DebugSymbol` 定位地址**：在有符号的 debug build 上，`DebugSymbol.fromAddress(addr)` 可以直接获取函数名和源码行号，极大提升效率。
