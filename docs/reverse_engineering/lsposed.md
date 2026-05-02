# LSPosed

LSPosed 是 Android 平台最主流的 Xposed 框架现代实现，基于 Magisk 运行，通过在 Zygote 进程中注入钩子，允许模块在不修改 APK 的情况下动态修改任意应用的行为。与旧版 Xposed Framework 相比，LSPosed 支持 Android 8.1–15，兼容性好、作用域隔离、隐蔽性强。

**核心概念：**

| 概念 | 说明 |
|------|------|
| **Xposed API** | 钩子接口，通过 `XposedHelpers` 等工具类 hook Java 方法 |
| **IXposedHookLoadPackage** | 最常用的入口接口，在目标包加载时触发 |
| **XC_MethodHook** | 方法钩子回调，在方法执行前/后插入自定义逻辑 |
| **XposedBridge** | 底层桥接类，提供 `hookMethod`、`log` 等核心方法 |
| **作用域（Scope）** | 模块生效的目标应用列表，由用户在 LSPosed 管理器中配置 |
| **Zygote** | Android 所有应用进程的父进程，LSPosed 在此注入钩子 |
| **Magisk Module** | LSPosed 以 Magisk 模块形式安装，依赖 Magisk/KernelSU/APatch |

---

## 环境准备

### 前置条件

| 条件 | 说明 |
|------|------|
| Root 权限 | 需要 Magisk（≥24.0）、KernelSU 或 APatch 之一 |
| Android 版本 | 8.1 ~ 15（API 27~35） |
| 架构 | arm64-v8a、armeabi-v7a、x86、x86_64 均支持 |

### 安装 LSPosed

**Magisk 方式（最常见）：**

1. 确认已安装 Magisk 并获得 Root
2. 在 [LSPosed Releases](https://github.com/LSPosed/LSPosed/releases) 下载最新 `LSPosed-v*-*-zygisk-release.zip`
3. Magisk 管理器 → 模块 → 从本地安装 → 选择 zip 文件
4. 重启设备
5. 通知栏或拨号盘输入 `*#*#5776733#*#*` 打开 LSPosed 管理器

**KernelSU / APatch 方式：**

下载对应的 `LSPosed-v*-*-zygisk-release.zip`，在对应管理器中安装模块，步骤相同。

### 开发环境

```
Android Studio（最新稳定版）
JDK 11 或 17
Android SDK（API 27+）
```

---

## 模块开发基础

### 项目结构

```
my-xposed-module/
├── app/
│   ├── src/main/
│   │   ├── java/com/example/module/
│   │   │   ├── HookEntry.java          # 钩子入口（实现 IXposedHookLoadPackage）
│   │   │   └── hooks/
│   │   │       ├── TargetAppHook.java
│   │   │       └── SystemHook.java
│   │   ├── assets/
│   │   │   └── xposed_init             # 声明入口类（必须）
│   │   └── res/
│   │       └── values/
│   │           └── strings.xml
│   └── build.gradle
└── build.gradle
```

### build.gradle 配置

```groovy
// app/build.gradle
plugins {
    id 'com.android.application'
}

android {
    compileSdk 35
    defaultConfig {
        applicationId "com.example.module"
        minSdk 27
        targetSdk 35
    }
}

dependencies {
    // XposedBridgeApi：compileOnly（不打包进 APK，运行时由 LSPosed 提供）
    compileOnly 'de.robv.android.xposed:api:82'
    compileOnly 'de.robv.android.xposed:api:82:sources'
}
```

```kotlin
// build.gradle (Kotlin DSL)
dependencies {
    compileOnly("de.robv.android.xposed:api:82")
}
```

**Gradle 仓库配置（`settings.gradle`）：**

```groovy
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        // XposedBridgeApi 仓库
        maven { url 'https://api.xposed.info/' }
    }
}
```

### AndroidManifest.xml 元数据

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application
        android:allowBackup="false"
        android:label="@string/app_name">

        <!-- 声明这是 Xposed 模块 -->
        <meta-data
            android:name="xposedmodule"
            android:value="true" />

        <!-- 模块描述（显示在 LSPosed 管理器中）-->
        <meta-data
            android:name="xposeddescription"
            android:value="这是一个示例 Xposed 模块" />

        <!-- 支持的最低 Xposed API 版本 -->
        <meta-data
            android:name="xposedminversion"
            android:value="82" />
    </application>
</manifest>
```

### xposed_init 文件

```
# app/src/main/assets/xposed_init
# 每行一个入口类的完整类名，支持多个入口
com.example.module.HookEntry
```

---

## 核心 API 详解

### IXposedHookLoadPackage — 包加载钩子

```java
import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.XposedBridge;
import de.robv.android.xposed.callbacks.XC_LoadPackage;

public class HookEntry implements IXposedHookLoadPackage {

    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) {
        // lpparam 包含目标进程的所有信息
        String packageName = lpparam.packageName;   // 包名
        ClassLoader classLoader = lpparam.classLoader; // 类加载器

        XposedBridge.log("加载包：" + packageName);

        // 只处理目标应用
        if (!packageName.equals("com.target.app")) return;

        hookTargetApp(classLoader);
    }

    private void hookTargetApp(ClassLoader classLoader) {
        // ... hook 逻辑
    }
}
```

### IXposedHookZygoteInit — Zygote 初始化钩子

```java
import de.robv.android.xposed.IXposedHookZygoteInit;
import de.robv.android.xposed.XSharedPreferences;

public class HookEntry implements IXposedHookZygoteInit {

    public static XSharedPreferences prefs;

    @Override
    public void initZygote(StartupParam startupParam) {
        // 在 Zygote 进程启动时执行（早于应用启动）
        // 适合初始化全局配置
        prefs = new XSharedPreferences("com.example.module", "config");
        prefs.makeWorldReadable();
    }
}
```

### IXposedHookInitPackageResources — 资源钩子

```java
import de.robv.android.xposed.IXposedHookInitPackageResources;
import de.robv.android.xposed.callbacks.XC_InitPackageResources;

public class HookEntry implements IXposedHookInitPackageResources {

    @Override
    public void handleInitPackageResources(
            XC_InitPackageResources.InitPackageResourcesParam resparam) {

        if (!resparam.packageName.equals("com.target.app")) return;

        // 替换目标应用的字符串资源
        resparam.res.setReplacement(
            "com.target.app", "string", "app_name", "我的修改版"
        );

        // 替换颜色资源
        resparam.res.setReplacement(
            "com.target.app", "color", "primary_color", 0xFF_FF0000
        );
    }
}
```

---

## XposedHelpers — 方法钩子

`XposedHelpers` 是最常用的工具类，提供通过类名/方法名查找并钩住方法的便捷 API。

### hookMethod — 钩住普通方法

```java
import de.robv.android.xposed.XC_MethodHook;
import de.robv.android.xposed.XposedHelpers;

// 语法：findAndHookMethod(类名或Class对象, classLoader, 方法名, [参数类型...], 回调)
XposedHelpers.findAndHookMethod(
    "com.target.app.LoginActivity",          // 目标类全名
    classLoader,
    "checkPassword",                          // 方法名
    String.class,                             // 参数类型（按顺序列出所有参数）
    new XC_MethodHook() {

        @Override
        protected void beforeHookedMethod(MethodHookParam param) {
            // 方法执行前调用
            // param.args[0] 是第一个参数（String password）
            String password = (String) param.args[0];
            XposedBridge.log("密码：" + password);

            // 修改参数
            param.args[0] = "modified_password";

            // 提前返回（跳过原方法执行）
            param.setResult(true);
        }

        @Override
        protected void afterHookedMethod(MethodHookParam param) {
            // 方法执行后调用
            // 获取返回值
            Boolean result = (Boolean) param.getResult();
            XposedBridge.log("checkPassword 返回：" + result);

            // 修改返回值
            param.setResult(true);

            // 如果有异常
            if (param.hasThrowable()) {
                XposedBridge.log("方法抛出异常：" + param.getThrowable());
                param.setThrowable(null); // 吞掉异常
            }
        }
    }
);
```

### hookConstructor — 钩住构造函数

```java
XposedHelpers.findAndHookConstructor(
    "com.target.app.User",
    classLoader,
    String.class, int.class,    // 构造函数参数类型
    new XC_MethodHook() {
        @Override
        protected void afterHookedMethod(MethodHookParam param) {
            // param.thisObject 是新创建的对象实例
            Object user = param.thisObject;
            XposedBridge.log("User 被创建：" + param.args[0]);

            // 修改对象字段
            XposedHelpers.setObjectField(user, "role", "admin");
        }
    }
);
```

### XC_MethodReplacement — 完全替换方法

```java
import de.robv.android.xposed.XC_MethodReplacement;

XposedHelpers.findAndHookMethod(
    "com.target.app.LicenseChecker",
    classLoader,
    "isLicenseValid",
    new XC_MethodReplacement() {
        @Override
        protected Object replaceHookedMethod(MethodHookParam param) {
            // 完全替换原方法，直接返回值
            return true;
        }
    }
);
```

### 字段操作

```java
// 获取对象字段
Object value = XposedHelpers.getObjectField(obj, "fieldName");
int    intVal = XposedHelpers.getIntField(obj, "intField");
boolean boolVal = XposedHelpers.getBooleanField(obj, "boolField");

// 设置对象字段
XposedHelpers.setObjectField(obj, "fieldName", newValue);
XposedHelpers.setIntField(obj, "intField", 42);
XposedHelpers.setBooleanField(obj, "boolField", true);

// 访问静态字段
Object staticVal = XposedHelpers.getStaticObjectField(clazz, "STATIC_FIELD");
XposedHelpers.setStaticObjectField(clazz, "STATIC_FIELD", newValue);

// 调用私有方法
Object result = XposedHelpers.callMethod(obj, "privateMethod", arg1, arg2);
Object result = XposedHelpers.callStaticMethod(clazz, "staticMethod", arg1);
```

### 查找类与方法

```java
// 查找类
Class<?> clazz = XposedHelpers.findClass("com.target.app.SomeClass", classLoader);

// 查找方法（不立即 hook，只获取 Method 对象）
Method method = XposedHelpers.findMethodExact(
    clazz, "methodName", String.class, int.class
);

// 通过 XposedBridge 直接 hook Method 对象
XposedBridge.hookMethod(method, new XC_MethodHook() { ... });

// 查找所有同名方法（方法重载）
XposedHelpers.findAndHookMethod(clazz, "overloadedMethod",
    XposedHelpers.findMethodsByName(clazz, "overloadedMethod")[0],
    new XC_MethodHook() { ... }
);
```

---

## 完整 Hook 示例

### 示例一：绕过 SSL Certificate Pinning

```java
public class SSLUnpinningHook implements IXposedHookLoadPackage {

    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) {
        // Hook OkHttp CertificatePinner
        try {
            XposedHelpers.findAndHookMethod(
                "okhttp3.CertificatePinner",
                lpparam.classLoader,
                "check",
                String.class,
                List.class,
                new XC_MethodReplacement() {
                    @Override
                    protected Object replaceHookedMethod(MethodHookParam param) {
                        // 什么都不做，跳过证书校验
                        return null;
                    }
                }
            );
        } catch (Throwable e) {
            XposedBridge.log("OkHttp hook 失败：" + e.getMessage());
        }

        // Hook 系统 TrustManager
        try {
            XposedHelpers.findAndHookMethod(
                "javax.net.ssl.TrustManagerFactory",
                lpparam.classLoader,
                "getTrustManagers",
                new XC_MethodReplacement() {
                    @Override
                    protected Object replaceHookedMethod(MethodHookParam param) throws Throwable {
                        return new TrustManager[]{createTrustAllManager()};
                    }
                }
            );
        } catch (Throwable e) {
            XposedBridge.log("TrustManager hook 失败：" + e.getMessage());
        }
    }

    private TrustManager createTrustAllManager() {
        return new X509TrustManager() {
            @Override
            public void checkClientTrusted(X509Certificate[] chain, String authType) {}
            @Override
            public void checkServerTrusted(X509Certificate[] chain, String authType) {}
            @Override
            public X509Certificate[] getAcceptedIssuers() { return new X509Certificate[0]; }
        };
    }
}
```

### 示例二：Hook Activity 生命周期（打印所有 Activity 跳转）

```java
public class ActivityTrackerHook implements IXposedHookLoadPackage {

    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) {
        XposedHelpers.findAndHookMethod(
            "android.app.Activity",
            lpparam.classLoader,
            "onCreate",
            Bundle.class,
            new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) {
                    Activity activity = (Activity) param.thisObject;
                    XposedBridge.log("[Activity] onCreate: "
                        + activity.getClass().getName()
                        + " | Intent: " + activity.getIntent());
                }
            }
        );

        // 同时 hook onResume
        XposedHelpers.findAndHookMethod(
            "android.app.Activity",
            lpparam.classLoader,
            "onResume",
            new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) {
                    Activity activity = (Activity) param.thisObject;
                    XposedBridge.log("[Activity] onResume: "
                        + activity.getClass().getName());
                }
            }
        );
    }
}
```

### 示例三：修改 SharedPreferences 返回值

```java
public class PrefsHook implements IXposedHookLoadPackage {

    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) {
        if (!lpparam.packageName.equals("com.target.app")) return;

        // Hook SharedPreferences.getBoolean
        XposedHelpers.findAndHookMethod(
            "android.app.SharedPreferencesImpl",
            lpparam.classLoader,
            "getBoolean",
            String.class, boolean.class,
            new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) {
                    String key = (String) param.args[0];
                    // 强制 VIP 开关返回 true
                    if ("is_vip".equals(key) || "premium_user".equals(key)) {
                        param.setResult(true);
                    }
                }
            }
        );
    }
}
```

### 示例四：Hook 系统应用（位置伪造）

```java
public class FakeLocationHook implements IXposedHookLoadPackage {

    private static final double FAKE_LAT = 39.9042;   // 北京天安门
    private static final double FAKE_LNG = 116.4074;

    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) {
        // Hook Location.getLatitude
        XposedHelpers.findAndHookMethod(
            "android.location.Location",
            lpparam.classLoader,
            "getLatitude",
            new XC_MethodReplacement() {
                @Override
                protected Object replaceHookedMethod(MethodHookParam param) {
                    return FAKE_LAT;
                }
            }
        );

        XposedHelpers.findAndHookMethod(
            "android.location.Location",
            lpparam.classLoader,
            "getLongitude",
            new XC_MethodReplacement() {
                @Override
                protected Object replaceHookedMethod(MethodHookParam param) {
                    return FAKE_LNG;
                }
            }
        );
    }
}
```

---

## Kotlin 写法

LSPosed 模块完全支持 Kotlin，推荐使用。

```kotlin
import de.robv.android.xposed.IXposedHookLoadPackage
import de.robv.android.xposed.XC_MethodHook
import de.robv.android.xposed.XposedBridge
import de.robv.android.xposed.XposedHelpers
import de.robv.android.xposed.callbacks.XC_LoadPackage

class HookEntry : IXposedHookLoadPackage {

    override fun handleLoadPackage(lpparam: XC_LoadPackage.LoadPackageParam) {
        if (lpparam.packageName != "com.target.app") return

        hookLoginCheck(lpparam.classLoader)
    }

    private fun hookLoginCheck(classLoader: ClassLoader) {
        XposedHelpers.findAndHookMethod(
            "com.target.app.LoginManager",
            classLoader,
            "isLoggedIn",
            object : XC_MethodHook() {
                override fun afterHookedMethod(param: MethodHookParam) {
                    // Kotlin 风格：直接赋值
                    param.result = true
                }
            }
        )
    }
}
```

### 封装工具扩展（推荐）

```kotlin
// utils/XposedExtensions.kt
import de.robv.android.xposed.XC_MethodHook
import de.robv.android.xposed.XC_MethodReplacement
import de.robv.android.xposed.XposedHelpers

fun ClassLoader.findClass(name: String): Class<*> =
    XposedHelpers.findClass(name, this)

fun Class<*>.hookMethod(
    methodName: String,
    vararg paramTypes: Any,
    before: ((XC_MethodHook.MethodHookParam) -> Unit)? = null,
    after:  ((XC_MethodHook.MethodHookParam) -> Unit)? = null
) {
    XposedHelpers.findAndHookMethod(
        this, methodName, *paramTypes,
        object : XC_MethodHook() {
            override fun beforeHookedMethod(param: MethodHookParam) { before?.invoke(param) }
            override fun afterHookedMethod(param: MethodHookParam)  { after?.invoke(param)  }
        }
    )
}

fun Class<*>.replaceMethod(
    methodName: String,
    vararg paramTypes: Any,
    replacement: (XC_MethodHook.MethodHookParam) -> Any?
) {
    XposedHelpers.findAndHookMethod(
        this, methodName, *paramTypes,
        object : XC_MethodReplacement() {
            override fun replaceHookedMethod(param: MethodHookParam) = replacement(param)
        }
    )
}

// 使用
class HookEntry : IXposedHookLoadPackage {
    override fun handleLoadPackage(lpparam: XC_LoadPackage.LoadPackageParam) {
        if (lpparam.packageName != "com.target.app") return

        val clazz = lpparam.classLoader.findClass("com.target.app.PaymentManager")

        // 替换付款校验
        clazz.replaceMethod("isPremiumUser") { true }

        // before + after
        clazz.hookMethod("purchase", String.class,
            before = { param ->
                XposedBridge.log("即将购买：${param.args[0]}")
            },
            after = { param ->
                XposedBridge.log("购买结果：${param.result}")
            }
        )
    }
}
```

---

## 模块配置与 SharedPreferences

### XSharedPreferences（推荐读取配置）

```java
// 在 Zygote 初始化阶段读取配置（跨进程安全）
public class HookEntry implements IXposedHookZygoteInit, IXposedHookLoadPackage {

    private static XSharedPreferences prefs;

    @Override
    public void initZygote(StartupParam startupParam) {
        prefs = new XSharedPreferences("com.example.module");
        prefs.makeWorldReadable();
    }

    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) {
        if (!lpparam.packageName.equals("com.target.app")) return;

        // 读取配置决定是否 hook
        prefs.reload();   // 刷新配置（避免缓存）
        boolean enabled = prefs.getBoolean("hook_enabled", true);
        if (!enabled) return;

        String targetMethod = prefs.getString("target_method", "checkLicense");
        // ... 根据配置动态 hook
    }
}
```

### 模块设置界面（Activity）

```kotlin
// 普通的 Android Activity，用于用户配置
class SettingsActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_settings)

        val prefs = getSharedPreferences("config", MODE_WORLD_READABLE) // 让其他进程可读

        // 读取开关状态
        val switch = findViewById<SwitchCompat>(R.id.switch_hook_enabled)
        switch.isChecked = prefs.getBoolean("hook_enabled", true)

        switch.setOnCheckedChangeListener { _, checked ->
            prefs.edit().putBoolean("hook_enabled", checked).apply()
        }
    }
}
```

---

## 多入口 / 多目标 Hook 组织模式

```java
// HookEntry.java — 主入口，分发到各子 Hook
public class HookEntry implements IXposedHookLoadPackage {

    private static final Map<String, BaseHook> HOOKS = new HashMap<String, BaseHook>() {{
        put("com.target.app",          new TargetAppHook());
        put("com.another.target",      new AnotherAppHook());
        put("android",                 new SystemFrameworkHook());
        put("com.android.settings",   new SettingsHook());
    }};

    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) {
        BaseHook hook = HOOKS.get(lpparam.packageName);
        if (hook != null) {
            try {
                hook.handleLoadPackage(lpparam);
            } catch (Throwable e) {
                XposedBridge.log("Hook 失败 [" + lpparam.packageName + "]: " + e);
            }
        }
    }
}

// BaseHook.java — 所有子 Hook 的基类
public abstract class BaseHook {
    public abstract void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam);

    protected void log(String msg) {
        XposedBridge.log("[MyModule] " + msg);
    }

    // 安全 hook，捕获 ClassNotFound 等异常
    protected void safeHook(Runnable hookAction) {
        try {
            hookAction.run();
        } catch (Throwable e) {
            log("Hook 异常：" + e);
        }
    }
}
```

---

## 调试技巧

### 查看日志

```shell
# LSPosed 日志（包含所有模块的 XposedBridge.log 输出）
adb logcat -s "Xposed"
adb logcat -s "LSPosed"

# 查看指定包的日志
adb logcat --pid=$(adb shell pidof com.target.app)

# 在 LSPosed 管理器中查看日志
# 管理器 → 日志 → 查看/导出

# 使用 XposedBridge.log 在代码中打日志
XposedBridge.log("TAG: " + variable);
// 输出格式：[LSPosed] TAG: value
```

### 常见问题排查

**模块不生效：**
```
1. 确认模块已在 LSPosed 管理器中启用
2. 确认目标应用已添加到模块作用域
3. 重启设备（改动 xposed_init 或 AndroidManifest 必须重启）
4. 检查 assets/xposed_init 路径和类名是否正确
5. 检查 Logcat 是否有 ClassNotFoundException
```

**ClassNotFoundException：**
```java
// 原因：目标类使用了自定义 ClassLoader（如热修复框架）
// 解决：延迟 hook，等目标类被加载后再 hook
XposedHelpers.findAndHookMethod(
    "dalvik.system.BaseDexClassLoader",
    lpparam.classLoader,
    "findClass",
    String.class,
    new XC_MethodHook() {
        @Override
        protected void afterHookedMethod(MethodHookParam param) {
            String className = (String) param.args[0];
            if ("com.target.app.HiddenClass".equals(className)) {
                Class<?> clazz = (Class<?>) param.getResult();
                if (clazz != null) {
                    // 此时目标类已被加载，可以 hook
                    hookTargetClass(clazz);
                }
            }
        }
    }
);
```

**方法签名找不到（NoSuchMethodError）：**
```shell
# 使用 jadx 或 dex2jar 反编译 APK 查看真实方法签名
# 查看方法是否是混淆后的名称
jadx -d output/ target.apk

# 使用 adb shell 查看方法列表
adb shell
cd /data/app/com.target.app-*/base.apk
# 配合 frida 动态枚举方法
```

**参数类型匹配失败：**
```java
// 错误：基本类型与包装类不能混用
XposedHelpers.findAndHookMethod(clazz, "method", Integer.class, ...); // ❌

// 正确：基本类型用 int.class，不是 Integer.class
XposedHelpers.findAndHookMethod(clazz, "method", int.class, ...);    // ✅

// 内部类的写法（用 $ 分隔）
XposedHelpers.findClass("com.target.app.Outer$Inner", classLoader);
```

---

## 与 Frida 对比

| 维度 | LSPosed (Xposed) | Frida |
|------|----------------|-------|
| **运行方式** | 持久化，设备重启后仍有效 | 需要每次手动注入（或配合 frida-server）|
| **Hook 语言** | Java / Kotlin | JavaScript / Python |
| **目标范围** | Java 层（部分 Native 通过反射）| Java + Native（更底层）|
| **Native Hook** | 需要额外库（如 SandHook）| 原生支持（Interceptor）|
| **检测风险** | 较易被检测（Xposed 特征明显）| 灵活，更难检测（但不安全的用法仍可被检测）|
| **适用场景** | 长期功能修改、模块化分发 | 动态分析、临时调试、逆向研究 |
| **上手难度** | 中（需要写 APK）| 低（脚本即用）|

---

## 反检测

部分应用会检测 Xposed / LSPosed 的存在，常见检测点和绕过方法：

### 常见检测方法

```java
// 1. 检查包名
PackageManager pm = context.getPackageManager();
pm.getPackageInfo("org.lsposed.manager", 0);      // LSPosed 管理器包名

// 2. 检查栈帧（调用栈中是否含 Xposed 类）
for (StackTraceElement e : Thread.currentThread().getStackTrace()) {
    if (e.getClassName().contains("de.robv.android.xposed")) {
        // 检测到 Xposed
    }
}

// 3. 读取 /proc/self/maps 查找 Xposed 相关 so
// 4. 检查 /data/dalvik-cache/ 中的 oat 是否被修改
// 5. 检查特定系统属性
```

### 绕过策略

```java
// 1. Hook PackageManager.getPackageInfo，让其对 LSPosed 相关包名返回异常
XposedHelpers.findAndHookMethod(
    "android.app.ApplicationPackageManager",
    lpparam.classLoader,
    "getPackageInfo",
    String.class, int.class,
    new XC_MethodHook() {
        @Override
        protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
            String pkgName = (String) param.args[0];
            if ("org.lsposed.manager".equals(pkgName)
                || "de.robv.android.xposed.installer".equals(pkgName)) {
                param.setThrowable(new PackageManager.NameNotFoundException(pkgName));
            }
        }
    }
);

// 2. Hook Thread.getStackTrace，过滤 Xposed 栈帧
XposedHelpers.findAndHookMethod(
    "java.lang.Thread",
    lpparam.classLoader,
    "getStackTrace",
    new XC_MethodHook() {
        @Override
        protected void afterHookedMethod(MethodHookParam param) {
            StackTraceElement[] stack = (StackTraceElement[]) param.getResult();
            param.setResult(filterXposedFrames(stack));
        }
    }
);

private StackTraceElement[] filterXposedFrames(StackTraceElement[] stack) {
    return Arrays.stream(stack)
        .filter(e -> !e.getClassName().contains("de.robv.android.xposed")
                  && !e.getClassName().contains("LSPosed"))
        .toArray(StackTraceElement[]::new);
}
```

---

## 常用现成模块参考

| 模块 | 功能 | 仓库 |
|------|------|------|
| **LSPatch** | 无 Root 将 Xposed 模块注入 APK | `LSPosed/LSPatch` |
| **TrustMeAlready** | 禁用 SSL 证书验证（调试用）| `ViRb3/TrustMeAlready` |
| **Hide My Applist** | 隐藏已安装应用列表 | `Dr-TSNG/Hide-My-Applist` |
| **Thanox** | 系统管理增强（后台管理、权限控制）| `Tornaco/Thanox` |
| **CorePatch** | 禁用 APK 签名验证 | `LSPosed/CorePatch` |
| **QAuxiliary** | QQ/TIM 功能增强 | `cinit/QAuxiliary` |

---

## 完整模块示例（Kotlin）

```kotlin
// HookEntry.kt
class HookEntry : IXposedHookLoadPackage {

    override fun handleLoadPackage(lpparam: XC_LoadPackage.LoadPackageParam) {
        when (lpparam.packageName) {
            "com.example.target" -> TargetAppHooks(lpparam).apply()
            "android"            -> SystemHooks(lpparam).apply()
        }
    }
}

// TargetAppHooks.kt
class TargetAppHooks(private val lpparam: XC_LoadPackage.LoadPackageParam) {

    fun apply() {
        hookVipCheck()
        hookNetworkRequest()
        hookRootDetection()
    }

    private fun hookVipCheck() = runCatching {
        XposedHelpers.findAndHookMethod(
            "com.example.target.UserManager",
            lpparam.classLoader,
            "isVip",
            object : XC_MethodReplacement() {
                override fun replaceHookedMethod(param: MethodHookParam) = true
            }
        )
        XposedBridge.log("[MyModule] hookVipCheck 成功")
    }.onFailure {
        XposedBridge.log("[MyModule] hookVipCheck 失败: $it")
    }

    private fun hookNetworkRequest() = runCatching {
        XposedHelpers.findAndHookMethod(
            "okhttp3.OkHttpClient\$Builder",
            lpparam.classLoader,
            "build",
            object : XC_MethodHook() {
                override fun afterHookedMethod(param: MethodHookParam) {
                    XposedBridge.log("[MyModule] OkHttpClient 构建，拦截请求")
                }
            }
        )
    }.onFailure {
        XposedBridge.log("[MyModule] hookNetworkRequest 失败: $it")
    }

    private fun hookRootDetection() = runCatching {
        // 拦截 Runtime.exec 执行 su 命令
        XposedHelpers.findAndHookMethod(
            "java.lang.Runtime",
            lpparam.classLoader,
            "exec",
            String::class.java,
            object : XC_MethodHook() {
                override fun beforeHookedMethod(param: MethodHookParam) {
                    val cmd = param.args[0] as? String ?: return
                    if (cmd.contains("su") || cmd.contains("which su")) {
                        param.setThrowable(IOException("命令执行被拦截"))
                    }
                }
            }
        )
    }.onFailure {
        XposedBridge.log("[MyModule] hookRootDetection 失败: $it")
    }
}
```

---

## 最佳实践

1. **每个 hook 独立 try-catch**：一个 hook 失败不应导致整个模块崩溃，用 `runCatching { }` 或 `try-catch` 包裹每个 `findAndHookMethod` 调用。

2. **精确指定作用域**：在 LSPosed 管理器中只勾选真正需要 hook 的应用，避免勾选系统全局，防止不相关应用崩溃。

3. **优先 `afterHookedMethod`**：在 `after` 中修改返回值比在 `before` 中中断执行更安全，出问题时原方法仍能执行。

4. **避免 hook 过于底层的系统方法**：如 `Object.toString()`、`Log.d()` 等调用极频繁的方法，会显著影响系统性能。

5. **用 `XposedBridge.log` 代替 `Log.d`**：后者在某些系统版本中无法正常输出，且 `XposedBridge.log` 会在 LSPosed 管理器日志中留存。

6. **混淆处理**：目标 APK 经常混淆，不要硬编码方法名，应通过特征（方法签名、字段类型、字符串常量）动态查找目标。

7. **版本适配**：目标 APK 更新后混淆名可能变化，模块发布时要注明支持的 APK 版本，关键逻辑加版本判断。

8. **模块本身不申请不必要权限**：模块 APK 运行在 System 进程上下文中，权限管控比普通应用更严格，保持最小权限。
