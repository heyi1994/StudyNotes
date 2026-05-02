# React Native

React Native 是 Meta 开发的跨平台移动端框架，使用 React 语法编写，最终编译为原生组件（而非 WebView）。熟悉 Flutter 的开发者会发现很多相似之处：组件树、声明式 UI、Flexbox 布局，但底层渲染和开发体验有所不同。

| | React Native | Flutter |
|---|---|---|
| 语言 | TypeScript / JavaScript | Dart |
| 渲染 | 映射到平台原生组件 | 自绘引擎（Skia/Impeller） |
| 布局 | Flexbox（仅列方向） | 自定义布局系统 |
| 生态 | npm 生态，原生模块丰富 | pub.dev，相对封闭 |
| 热重载 | 支持（Fast Refresh） | 支持 |

---

## 安装与创建项目

推荐使用 **Expo**（托管工作流），开发体验最佳，无需配置原生环境即可启动：

```shell
npx create-expo-app@latest my-app
cd my-app
npx expo start
```

扫描终端中的二维码，在手机上通过 **Expo Go** App 预览；或按 `i` / `a` 启动模拟器。

### 裸 React Native（需配置原生环境）

```shell
npx @react-native-community/cli@latest init MyApp --template react-native-template-typescript
cd MyApp
npx react-native run-ios    # iOS
npx react-native run-android # Android
```

### 项目结构

```
my-app/
├── app/                  # Expo Router 页面（文件路由）
│   ├── _layout.tsx       # 根布局
│   ├── index.tsx         # 首页 /
│   └── (tabs)/
│       ├── _layout.tsx   # Tab 布局
│       ├── index.tsx     # Tab 1
│       └── explore.tsx   # Tab 2
├── components/           # 复用组件
├── hooks/                # 自定义 Hook
├── constants/            # 颜色、配置等常量
├── assets/               # 图片、字体等静态资源
├── app.json              # Expo 配置
└── tsconfig.json
```

---

## 核心组件

React Native 没有 DOM，不能使用 `div`、`span`、`p` 等 HTML 标签，必须使用 RN 提供的原生组件。

| RN 组件 | 对应 Flutter Widget | 说明 |
|---------|-------------------|------|
| `View` | `Container` / `Box` | 布局容器 |
| `Text` | `Text` | 文本（所有文本必须在 Text 内） |
| `Image` | `Image` | 图片 |
| `ScrollView` | `SingleChildScrollView` | 可滚动容器 |
| `FlatList` | `ListView.builder` | 长列表（虚拟化） |
| `TextInput` | `TextField` | 输入框 |
| `TouchableOpacity` | `GestureDetector` / `InkWell` | 可点击容器（透明度反馈） |
| `Pressable` | `GestureDetector` | 更灵活的可点击容器 |
| `Modal` | `showDialog` | 弹窗 |
| `ActivityIndicator` | `CircularProgressIndicator` | 加载指示器 |
| `Switch` | `Switch` | 开关 |

```tsx
import { View, Text, Image, TouchableOpacity, StyleSheet } from "react-native"

export default function Card() {
  return (
    <View style={styles.card}>
      <Image
        source={{ uri: "https://example.com/avatar.png" }}
        style={styles.avatar}
      />
      <View style={styles.info}>
        <Text style={styles.name}>张三</Text>
        <Text style={styles.bio} numberOfLines={2}>
          这是一段个人简介，超过两行会显示省略号
        </Text>
      </View>
      <TouchableOpacity style={styles.button} onPress={() => console.log("pressed")}>
        <Text style={styles.buttonText}>关注</Text>
      </TouchableOpacity>
    </View>
  )
}

const styles = StyleSheet.create({
  card:       { flexDirection: "row", padding: 16, backgroundColor: "#fff", borderRadius: 12 },
  avatar:     { width: 48, height: 48, borderRadius: 24 },
  info:       { flex: 1, marginHorizontal: 12 },
  name:       { fontSize: 16, fontWeight: "600", color: "#111" },
  bio:        { fontSize: 14, color: "#666", marginTop: 4 },
  button:     { backgroundColor: "#3b82f6", paddingHorizontal: 16, paddingVertical: 8, borderRadius: 8 },
  buttonText: { color: "#fff", fontSize: 14, fontWeight: "500" },
})
```

---

## StyleSheet 样式系统

RN 使用 JavaScript 对象描述样式，**不是** CSS（虽然属性名相同），有以下区别：

- 属性名用**驼峰命名**：`backgroundColor`、`fontSize`、`borderRadius`
- 尺寸单位是**密度无关像素（dp）**，不需要写 `px`
- 不支持所有 CSS 属性（如 `box-shadow` → 用 `shadow*` 系列替代）
- Flexbox 默认主轴为**列方向**（`flexDirection: "column"`），与 Web 相反

```typescript
import { StyleSheet, Platform } from "react-native"

const styles = StyleSheet.create({
  // 基础样式
  container: {
    flex: 1,
    backgroundColor: "#f5f5f5",
    padding: 16,
  },

  // 文字
  title: {
    fontSize: 24,
    fontWeight: "bold",
    color: "#111827",
    letterSpacing: 0.5,
    lineHeight: 32,
  },

  // 阴影（iOS）
  card: {
    backgroundColor: "#fff",
    borderRadius: 12,
    padding: 16,
    shadowColor: "#000",
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.1,
    shadowRadius: 8,
    elevation: 4,  // Android 阴影
  },

  // 平台差异写法
  header: {
    paddingTop: Platform.OS === "ios" ? 50 : 24,
    ...Platform.select({
      ios:     { shadowColor: "#000", shadowOpacity: 0.1 },
      android: { elevation: 4 },
    }),
  },
})
```

### 动态样式

```tsx
// 样式数组（后面的覆盖前面的）
<View style={[styles.base, isActive && styles.active, { opacity: disabled ? 0.5 : 1 }]} />

// 内联样式（性能较差，不推荐频繁使用）
<Text style={{ color: isDark ? "#fff" : "#000" }}>文本</Text>
```

---

## Flexbox 布局

RN 的 Flexbox 与 Web 基本一致，但有两点核心差异：

1. **默认 `flexDirection: "column"`**（Web 默认是 row）
2. **默认 `alignContent: "flex-start"`**

```tsx
// 垂直居中（主轴是列方向）
<View style={{ flex: 1, justifyContent: "center", alignItems: "center" }}>

// 水平排列
<View style={{ flexDirection: "row", alignItems: "center", gap: 12 }}>
  <Icon />
  <Text>标签</Text>
</View>

// 子项均分
<View style={{ flexDirection: "row" }}>
  <View style={{ flex: 1 }}>左</View>
  <View style={{ flex: 2 }}>中（占 2/3）</View>
  <View style={{ flex: 1 }}>右</View>
</View>

// 底部固定
<View style={{ flex: 1 }}>
  <ScrollView style={{ flex: 1 }}>内容区域</ScrollView>
  <View>底部固定栏</View>
</View>
```

---

## 文本组件

```tsx
import { Text } from "react-native"

// 所有文字必须放在 Text 组件内，不能直接放在 View 里
<Text style={{ fontSize: 16, color: "#333" }}>普通文字</Text>

// 嵌套 Text（内联样式）
<Text style={{ fontSize: 16 }}>
  这是 <Text style={{ fontWeight: "bold" }}>加粗</Text> 文字
  和 <Text style={{ color: "red" }}>红色</Text> 文字
</Text>

// 限制行数
<Text numberOfLines={3} ellipsizeMode="tail">
  很长的文本内容...
</Text>

// 可选中文本
<Text selectable>可以长按选中的文字</Text>

// 可点击（简单场景，复杂场景用 Pressable）
<Text onPress={() => Linking.openURL("https://example.com")}>
  点击打开链接
</Text>
```

---

## 图片

```tsx
import { Image, ImageBackground } from "react-native"

// 网络图片（必须指定宽高）
<Image
  source={{ uri: "https://example.com/photo.jpg" }}
  style={{ width: 200, height: 150, borderRadius: 8 }}
  resizeMode="cover"   // cover | contain | stretch | repeat | center
/>

// 本地图片（自动获取尺寸）
import logo from "@/assets/logo.png"
<Image source={logo} style={{ width: 120, height: 40 }} />

// 背景图
<ImageBackground
  source={{ uri: "https://example.com/bg.jpg" }}
  style={{ width: "100%", height: 200 }}
  resizeMode="cover"
>
  <Text style={{ color: "#fff", padding: 16 }}>图片上的文字</Text>
</ImageBackground>
```

### Expo Image（推荐，功能更强）

```shell
npx expo install expo-image
```

```tsx
import { Image } from "expo-image"

<Image
  source="https://example.com/photo.jpg"
  style={{ width: 200, height: 150 }}
  contentFit="cover"
  placeholder={{ blurhash: "L6PZfSi_.AyE_3t7t7R**0o#DgR4" }}
  transition={300}  // 淡入动画
/>
```

---

## 列表

### FlatList（虚拟化长列表）

类似 Flutter 的 `ListView.builder`，只渲染可见区域的条目：

```tsx
import { FlatList, View, Text, RefreshControl } from "react-native"

interface Post {
  id: string
  title: string
  content: string
}

function PostList() {
  const [posts, setPosts] = useState<Post[]>([])
  const [refreshing, setRefreshing] = useState(false)
  const [loadingMore, setLoadingMore] = useState(false)

  async function handleRefresh() {
    setRefreshing(true)
    const data = await fetchPosts({ page: 1 })
    setPosts(data)
    setRefreshing(false)
  }

  async function handleLoadMore() {
    if (loadingMore) return
    setLoadingMore(true)
    const data = await fetchPosts({ page: Math.ceil(posts.length / 20) + 1 })
    setPosts(prev => [...prev, ...data])
    setLoadingMore(false)
  }

  return (
    <FlatList
      data={posts}
      keyExtractor={(item) => item.id}
      renderItem={({ item, index }) => (
        <View style={styles.item}>
          <Text style={styles.title}>{item.title}</Text>
          <Text>{item.content}</Text>
        </View>
      )}
      // 下拉刷新
      refreshControl={
        <RefreshControl refreshing={refreshing} onRefresh={handleRefresh} />
      }
      // 上拉加载更多
      onEndReached={handleLoadMore}
      onEndReachedThreshold={0.3}  // 距离底部 30% 时触发
      ListFooterComponent={loadingMore ? <ActivityIndicator /> : null}
      // 空状态
      ListEmptyComponent={<Text style={styles.empty}>暂无数据</Text>}
      // 头部
      ListHeaderComponent={<Text style={styles.header}>最新文章</Text>}
      // 性能优化
      removeClippedSubviews={true}
      maxToRenderPerBatch={10}
      windowSize={10}
      // 横向列表
      // horizontal
      // showsHorizontalScrollIndicator={false}
      // 多列网格（类似 GridView）
      // numColumns={2}
      // columnWrapperStyle={{ gap: 12 }}
    />
  )
}
```

### SectionList（分组列表）

类似 Flutter 的分组 ListView：

```tsx
import { SectionList, Text, View } from "react-native"

const sections = [
  { title: "今天", data: ["任务 A", "任务 B"] },
  { title: "明天", data: ["任务 C", "任务 D", "任务 E"] },
]

<SectionList
  sections={sections}
  keyExtractor={(item, index) => item + index}
  renderItem={({ item }) => (
    <View style={styles.item}><Text>{item}</Text></View>
  )}
  renderSectionHeader={({ section }) => (
    <Text style={styles.sectionHeader}>{section.title}</Text>
  )}
  stickySectionHeadersEnabled={true}  // 吸顶标题
/>
```

---

## 可点击组件

### Pressable（推荐）

```tsx
import { Pressable, Text } from "react-native"

// 样式函数，根据按压状态动态改变
<Pressable
  onPress={() => console.log("pressed")}
  onLongPress={() => console.log("long pressed")}
  style={({ pressed }) => [
    styles.button,
    pressed && { opacity: 0.7, transform: [{ scale: 0.97 }] },
  ]}
>
  {({ pressed }) => (
    <Text style={[styles.text, pressed && { color: "#999" }]}>
      按钮
    </Text>
  )}
</Pressable>
```

### TouchableOpacity

```tsx
import { TouchableOpacity } from "react-native"

<TouchableOpacity
  activeOpacity={0.7}   // 按下时的透明度，默认 0.2
  onPress={handlePress}
  disabled={isLoading}
>
  <Text>按钮</Text>
</TouchableOpacity>
```

---

## 输入框

```tsx
import { TextInput, View, Text } from "react-native"

function LoginForm() {
  const [email, setEmail] = useState("")
  const [password, setPassword] = useState("")
  const passwordRef = useRef<TextInput>(null)

  return (
    <View style={{ gap: 16 }}>
      <TextInput
        style={styles.input}
        value={email}
        onChangeText={setEmail}          // 类似 Flutter 的 onChanged
        placeholder="邮箱"
        placeholderTextColor="#9ca3af"
        keyboardType="email-address"      // 弹出邮箱键盘
        autoCapitalize="none"             // 不自动大写
        autoCorrect={false}
        returnKeyType="next"              // 键盘回车键显示"下一步"
        onSubmitEditing={() => passwordRef.current?.focus()}
      />
      <TextInput
        ref={passwordRef}
        style={styles.input}
        value={password}
        onChangeText={setPassword}
        placeholder="密码"
        placeholderTextColor="#9ca3af"
        secureTextEntry={true}           // 密码模式
        returnKeyType="done"
        onSubmitEditing={handleLogin}
      />
    </View>
  )
}

const styles = StyleSheet.create({
  input: {
    borderWidth: 1,
    borderColor: "#d1d5db",
    borderRadius: 8,
    paddingHorizontal: 12,
    paddingVertical: 10,
    fontSize: 16,
    color: "#111827",
    backgroundColor: "#fff",
  },
})
```

### keyboardType 常用值

| 值 | 说明 |
|----|------|
| `default` | 默认键盘 |
| `numeric` | 数字键盘 |
| `decimal-pad` | 带小数点的数字键盘 |
| `email-address` | 邮箱键盘（含 @ 和 .） |
| `phone-pad` | 电话键盘 |
| `url` | URL 键盘 |

---

## ScrollView

```tsx
import { ScrollView, KeyboardAvoidingView, Platform } from "react-native"

// 基本滚动容器
<ScrollView
  contentContainerStyle={{ padding: 16, gap: 12 }}
  showsVerticalScrollIndicator={false}
  bounces={true}          // iOS 回弹效果
>
  {/* 内容 */}
</ScrollView>

// 横向滚动（如标签栏）
<ScrollView
  horizontal
  showsHorizontalScrollIndicator={false}
  contentContainerStyle={{ gap: 8, paddingHorizontal: 16 }}
>
  {tags.map(tag => <TagChip key={tag} label={tag} />)}
</ScrollView>

// 键盘遮挡处理（表单页面常用）
<KeyboardAvoidingView
  style={{ flex: 1 }}
  behavior={Platform.OS === "ios" ? "padding" : "height"}
>
  <ScrollView keyboardShouldPersistTaps="handled">
    {/* 表单内容 */}
  </ScrollView>
</KeyboardAvoidingView>
```

---

## 导航（React Navigation）

React Native 最主流的导航库。

```shell
npx expo install @react-navigation/native
npx expo install @react-navigation/native-stack
npx expo install @react-navigation/bottom-tabs
npx expo install react-native-screens react-native-safe-area-context
```

### Stack 导航

```tsx
// App.tsx
import { NavigationContainer } from "@react-navigation/native"
import { createNativeStackNavigator } from "@react-navigation/native-stack"

// 定义路由参数类型
type RootStackParamList = {
  Home: undefined
  Profile: { userId: string; name: string }
  Settings: undefined
}

const Stack = createNativeStackNavigator<RootStackParamList>()

export default function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator
        initialRouteName="Home"
        screenOptions={{
          headerStyle: { backgroundColor: "#fff" },
          headerTintColor: "#111",
          headerTitleStyle: { fontWeight: "600" },
          animation: "slide_from_right",
        }}
      >
        <Stack.Screen
          name="Home"
          component={HomeScreen}
          options={{ title: "首页" }}
        />
        <Stack.Screen
          name="Profile"
          component={ProfileScreen}
          options={({ route }) => ({ title: route.params.name })}
        />
        <Stack.Screen
          name="Settings"
          component={SettingsScreen}
          options={{
            presentation: "modal",   // 从底部弹出
            title: "设置",
          }}
        />
      </Stack.Navigator>
    </NavigationContainer>
  )
}
```

### 页面跳转与参数传递

```tsx
import { useNavigation, useRoute } from "@react-navigation/native"
import type { NativeStackNavigationProp } from "@react-navigation/native-stack"
import type { RouteProp } from "@react-navigation/native"

type HomeScreenNavigationProp = NativeStackNavigationProp<RootStackParamList, "Home">

function HomeScreen() {
  const navigation = useNavigation<HomeScreenNavigationProp>()

  return (
    <View>
      {/* 导航到新页面 */}
      <Button title="去个人页" onPress={() =>
        navigation.navigate("Profile", { userId: "1", name: "张三" })
      } />

      {/* 替换当前页 */}
      <Button title="替换" onPress={() =>
        navigation.replace("Profile", { userId: "2", name: "李四" })
      } />

      {/* 返回 */}
      <Button title="返回" onPress={() => navigation.goBack()} />

      {/* 返回到指定页 */}
      <Button title="回首页" onPress={() => navigation.popTo("Home")} />

      {/* 设置导航栏按钮 */}
      <Button title="设置标题" onPress={() =>
        navigation.setOptions({ title: "新标题" })
      } />
    </View>
  )
}

// 接收参数
type ProfileRouteProp = RouteProp<RootStackParamList, "Profile">

function ProfileScreen() {
  const route = useRoute<ProfileRouteProp>()
  const { userId, name } = route.params

  return <Text>用户 {name}（ID: {userId}）</Text>
}
```

### Tab 导航

```tsx
import { createBottomTabNavigator } from "@react-navigation/bottom-tabs"
import { Ionicons } from "@expo/vector-icons"

const Tab = createBottomTabNavigator()

function TabNavigator() {
  return (
    <Tab.Navigator
      screenOptions={({ route }) => ({
        tabBarIcon: ({ focused, color, size }) => {
          const icons: Record<string, string> = {
            Home:    focused ? "home" : "home-outline",
            Search:  focused ? "search" : "search-outline",
            Profile: focused ? "person" : "person-outline",
          }
          return <Ionicons name={icons[route.name] as any} size={size} color={color} />
        },
        tabBarActiveTintColor: "#3b82f6",
        tabBarInactiveTintColor: "#9ca3af",
        tabBarStyle: { borderTopColor: "#e5e7eb" },
        headerShown: false,
      })}
    >
      <Tab.Screen name="Home" component={HomeScreen} options={{ title: "首页" }} />
      <Tab.Screen name="Search" component={SearchScreen} options={{ title: "搜索" }} />
      <Tab.Screen name="Profile" component={ProfileScreen} options={{ title: "我的" }} />
    </Tab.Navigator>
  )
}
```

### 嵌套导航

```tsx
// Stack 内嵌 Tab
const Stack = createNativeStackNavigator()

function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Main" component={TabNavigator} options={{ headerShown: false }} />
        <Stack.Screen name="Detail" component={DetailScreen} />
        <Stack.Screen name="Login" component={LoginScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  )
}
```

---

## Expo Router（文件路由，推荐）

Expo Router 基于文件系统路由，类似 Next.js，是 Expo 项目的首选导航方案：

```
app/
├── _layout.tsx            # 根布局（Stack 导航）
├── index.tsx              # /（首页）
├── (tabs)/
│   ├── _layout.tsx        # Tab 布局
│   ├── index.tsx          # /（Tab 1）
│   └── explore.tsx        # /explore（Tab 2）
├── profile/
│   ├── [id].tsx           # /profile/:id（动态路由）
│   └── index.tsx          # /profile
└── modal.tsx              # /modal
```

```tsx
// app/_layout.tsx
import { Stack } from "expo-router"

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      <Stack.Screen name="modal" options={{ presentation: "modal" }} />
    </Stack>
  )
}

// app/(tabs)/_layout.tsx
import { Tabs } from "expo-router"
import { Ionicons } from "@expo/vector-icons"

export default function TabLayout() {
  return (
    <Tabs screenOptions={{ tabBarActiveTintColor: "#3b82f6" }}>
      <Tabs.Screen
        name="index"
        options={{
          title: "首页",
          tabBarIcon: ({ color }) => <Ionicons name="home" size={24} color={color} />,
        }}
      />
      <Tabs.Screen
        name="explore"
        options={{
          title: "探索",
          tabBarIcon: ({ color }) => <Ionicons name="compass" size={24} color={color} />,
        }}
      />
    </Tabs>
  )
}
```

```tsx
// 导航
import { router, useLocalSearchParams, Link } from "expo-router"

// 声明式
<Link href="/profile/42">个人页</Link>
<Link href={{ pathname: "/profile/[id]", params: { id: "42" } }}>个人页</Link>

// 编程式
router.push("/profile/42")
router.push({ pathname: "/profile/[id]", params: { id: "42" } })
router.replace("/login")
router.back()

// 接收参数
function ProfilePage() {
  const { id } = useLocalSearchParams<{ id: string }>()
  return <Text>用户 ID：{id}</Text>
}
```

---

## 安全区域

```tsx
import { SafeAreaView } from "react-native-safe-area-context"
import { useSafeAreaInsets } from "react-native-safe-area-context"

// 方式一：SafeAreaView 包裹
function Screen() {
  return (
    <SafeAreaView style={{ flex: 1 }} edges={["top", "bottom"]}>
      <Content />
    </SafeAreaView>
  )
}

// 方式二：手动获取 insets（更灵活）
function Screen() {
  const insets = useSafeAreaInsets()

  return (
    <View style={{ flex: 1, paddingTop: insets.top, paddingBottom: insets.bottom }}>
      <Content />
    </View>
  )
}
```

---

## 动画

### Animated API（内置）

```tsx
import { Animated, Easing } from "react-native"

function FadeIn({ children }: { children: React.ReactNode }) {
  const opacity = useRef(new Animated.Value(0)).current

  useEffect(() => {
    Animated.timing(opacity, {
      toValue: 1,
      duration: 400,
      easing: Easing.out(Easing.quad),
      useNativeDriver: true,  // 必须开启，使用原生线程执行动画
    }).start()
  }, [])

  return <Animated.View style={{ opacity }}>{children}</Animated.View>
}

// 序列动画
Animated.sequence([
  Animated.timing(scale, { toValue: 1.2, duration: 200, useNativeDriver: true }),
  Animated.timing(scale, { toValue: 1.0, duration: 200, useNativeDriver: true }),
]).start()

// 并行动画
Animated.parallel([
  Animated.timing(opacity, { toValue: 1, duration: 300, useNativeDriver: true }),
  Animated.timing(translateY, { toValue: 0, duration: 300, useNativeDriver: true }),
]).start()

// Spring 弹性动画
Animated.spring(scale, {
  toValue: 1,
  tension: 40,
  friction: 7,
  useNativeDriver: true,
}).start()

// 插值（将一个范围映射到另一个范围）
const rotate = opacity.interpolate({
  inputRange: [0, 1],
  outputRange: ["0deg", "360deg"],
})

<Animated.View style={{
  opacity,
  transform: [{ scale }, { rotate }],
}}>
```

### Reanimated 3（推荐，流畅度更高）

```shell
npx expo install react-native-reanimated
```

```tsx
import Animated, {
  useSharedValue,
  useAnimatedStyle,
  withTiming,
  withSpring,
  withRepeat,
  interpolate,
  runOnJS,
} from "react-native-reanimated"

function AnimatedCard() {
  const scale   = useSharedValue(1)
  const opacity = useSharedValue(1)

  // 动画样式（运行在 UI 线程）
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [{ scale: scale.value }],
    opacity: opacity.value,
  }))

  function handlePressIn() {
    scale.value   = withSpring(0.95)
    opacity.value = withTiming(0.8, { duration: 100 })
  }

  function handlePressOut() {
    scale.value   = withSpring(1)
    opacity.value = withTiming(1, { duration: 150 })
  }

  return (
    <Pressable onPressIn={handlePressIn} onPressOut={handlePressOut}>
      <Animated.View style={[styles.card, animatedStyle]}>
        <Text>点击我</Text>
      </Animated.View>
    </Pressable>
  )
}

// 循环动画（加载指示器）
function Spinner() {
  const rotation = useSharedValue(0)

  useEffect(() => {
    rotation.value = withRepeat(
      withTiming(360, { duration: 1000, easing: Easing.linear }),
      -1,   // 无限循环
    )
  }, [])

  const style = useAnimatedStyle(() => ({
    transform: [{ rotate: `${rotation.value}deg` }],
  }))

  return <Animated.View style={[styles.spinner, style]} />
}
```

---

## 手势（Gesture Handler）

```shell
npx expo install react-native-gesture-handler
```

```tsx
import { Gesture, GestureDetector } from "react-native-gesture-handler"
import Animated, { useSharedValue, useAnimatedStyle, withSpring } from "react-native-reanimated"

function DraggableCard() {
  const translateX = useSharedValue(0)
  const translateY = useSharedValue(0)
  const startX     = useSharedValue(0)
  const startY     = useSharedValue(0)

  const panGesture = Gesture.Pan()
    .onBegin(() => {
      startX.value = translateX.value
      startY.value = translateY.value
    })
    .onUpdate((e) => {
      translateX.value = startX.value + e.translationX
      translateY.value = startY.value + e.translationY
    })
    .onEnd(() => {
      translateX.value = withSpring(0)
      translateY.value = withSpring(0)
    })

  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
    ],
  }))

  return (
    <GestureDetector gesture={panGesture}>
      <Animated.View style={[styles.card, animatedStyle]}>
        <Text>拖动我</Text>
      </Animated.View>
    </GestureDetector>
  )
}
```

---

## 本地存储

### AsyncStorage（键值对存储）

```shell
npx expo install @react-native-async-storage/async-storage
```

```typescript
import AsyncStorage from "@react-native-async-storage/async-storage"

// 存储
await AsyncStorage.setItem("token", "abc123")
await AsyncStorage.setItem("user", JSON.stringify({ id: 1, name: "张三" }))

// 读取
const token = await AsyncStorage.getItem("token")
const raw   = await AsyncStorage.getItem("user")
const user  = raw ? JSON.parse(raw) : null

// 删除
await AsyncStorage.removeItem("token")

// 清空所有
await AsyncStorage.clear()

// 批量操作
await AsyncStorage.multiSet([
  ["key1", "value1"],
  ["key2", "value2"],
])
const values = await AsyncStorage.multiGet(["key1", "key2"])
```

### expo-secure-store（敏感数据，使用 Keychain/Keystore）

```shell
npx expo install expo-secure-store
```

```typescript
import * as SecureStore from "expo-secure-store"

// 存储（加密保存到系统安全存储）
await SecureStore.setItemAsync("jwt_token", token)

// 读取
const token = await SecureStore.getItemAsync("jwt_token")

// 删除
await SecureStore.deleteItemAsync("jwt_token")
```

---

## 常用 Expo API

### 相机

```shell
npx expo install expo-camera expo-image-picker
```

```tsx
import * as ImagePicker from "expo-image-picker"

async function pickImage() {
  // 请求权限
  const { status } = await ImagePicker.requestMediaLibraryPermissionsAsync()
  if (status !== "granted") return

  const result = await ImagePicker.launchImageLibraryAsync({
    mediaTypes: ["images"],
    allowsEditing: true,
    aspect: [1, 1],
    quality: 0.8,
  })

  if (!result.canceled) {
    const uri = result.assets[0].uri
    console.log("选择的图片：", uri)
  }
}

async function takePhoto() {
  const { status } = await ImagePicker.requestCameraPermissionsAsync()
  if (status !== "granted") return

  const result = await ImagePicker.launchCameraAsync({
    allowsEditing: true,
    quality: 0.8,
  })

  if (!result.canceled) {
    console.log("拍摄的照片：", result.assets[0].uri)
  }
}
```

### 通知

```shell
npx expo install expo-notifications
```

```typescript
import * as Notifications from "expo-notifications"

// 请求权限
const { status } = await Notifications.requestPermissionsAsync()

// 发送本地通知
await Notifications.scheduleNotificationAsync({
  content: {
    title: "提醒",
    body: "您有一条新消息",
    data: { screen: "Messages" },
  },
  trigger: { seconds: 5 },  // 5 秒后触发
})

// 推送 Token（用于服务端推送）
const token = await Notifications.getExpoPushTokenAsync({
  projectId: Constants.expoConfig?.extra?.eas?.projectId,
})
```

### 位置

```shell
npx expo install expo-location
```

```typescript
import * as Location from "expo-location"

const { status } = await Location.requestForegroundPermissionsAsync()
if (status !== "granted") return

const location = await Location.getCurrentPositionAsync({ accuracy: Location.Accuracy.High })
console.log(location.coords.latitude, location.coords.longitude)

// 持续监听位置变化
const subscription = await Location.watchPositionAsync(
  { accuracy: Location.Accuracy.Balanced, timeInterval: 5000 },
  (location) => console.log(location.coords),
)
// 取消监听
subscription.remove()
```

---

## 平台差异处理

```tsx
import { Platform } from "react-native"

// 判断平台
if (Platform.OS === "ios")     { /* iOS 逻辑 */ }
if (Platform.OS === "android") { /* Android 逻辑 */ }
if (Platform.OS === "web")     { /* Web 逻辑 */ }

// Platform.select（更简洁）
const styles = StyleSheet.create({
  header: {
    paddingTop: Platform.select({ ios: 50, android: 24, default: 0 }),
    ...Platform.select({
      ios: {
        shadowColor: "#000",
        shadowOffset: { width: 0, height: 1 },
        shadowOpacity: 0.1,
      },
      android: {
        elevation: 2,
      },
    }),
  },
})

// 平台特定文件（自动选择）
// Button.ios.tsx   → iOS 使用
// Button.android.tsx → Android 使用
// Button.tsx       → 兜底

// 版本判断
if (Platform.Version >= 14) { /* iOS 14+ 逻辑 */ }
```

---

## 网络请求

```typescript
// 基础 fetch
async function fetchPosts(): Promise<Post[]> {
  const res = await fetch("https://api.example.com/posts", {
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json",
    },
  })
  if (!res.ok) throw new Error(`HTTP ${res.status}`)
  return res.json()
}

// 上传图片（multipart/form-data）
async function uploadAvatar(uri: string) {
  const formData = new FormData()
  formData.append("avatar", {
    uri,
    type: "image/jpeg",
    name: "avatar.jpg",
  } as any)

  const res = await fetch("/api/upload", {
    method: "POST",
    body: formData,
    headers: { "Content-Type": "multipart/form-data" },
  })
  return res.json()
}
```

推荐搭配 **TanStack Query**（React Query）进行数据请求与缓存：

```shell
npm install @tanstack/react-query
```

```tsx
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query"

function PostList() {
  const { data, isLoading, error, refetch } = useQuery({
    queryKey: ["posts"],
    queryFn: fetchPosts,
    staleTime: 1000 * 60 * 5,  // 5 分钟内不重新请求
  })

  const queryClient = useQueryClient()
  const { mutate: deletePost } = useMutation({
    mutationFn: (id: number) => fetch(`/api/posts/${id}`, { method: "DELETE" }),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ["posts"] }),
  })

  if (isLoading) return <ActivityIndicator />

  return (
    <FlatList
      data={data}
      keyExtractor={(item) => String(item.id)}
      renderItem={({ item }) => (
        <Pressable onLongPress={() => deletePost(item.id)}>
          <Text>{item.title}</Text>
        </Pressable>
      )}
      refreshing={false}
      onRefresh={refetch}
    />
  )
}
```

---

## 调试

### Expo 开发工具

- 摇晃设备 或 按 `m` 键打开开发菜单
- `j` 键：打开 Chrome DevTools
- `r` 键：重新加载
- Expo Go App 内有日志面板

### React Native Debugger

推荐安装 **Flipper** 或使用 Chrome DevTools：

```shell
# 查看设备日志
npx react-native log-ios
npx react-native log-android
```

### 常见问题

```tsx
// 问题：Metro 缓存问题
npx expo start --clear

// 问题：Android 模拟器无法连接
adb reverse tcp:8081 tcp:8081

// 问题：键盘遮挡输入框
// 使用 KeyboardAvoidingView + ScrollView 解决

// 问题：列表性能差
// 确保 renderItem 使用 React.memo，keyExtractor 返回稳定唯一值
const renderItem = useCallback(({ item }: { item: Post }) => (
  <PostItem post={item} />
), [])

const PostItem = React.memo(({ post }: { post: Post }) => (
  <View>
    <Text>{post.title}</Text>
  </View>
))
```

---

## 常用第三方库速查

| 功能 | 推荐库 |
|------|--------|
| 导航 | `expo-router` / `@react-navigation/native` |
| 状态管理 | `zustand` / `@reduxjs/toolkit` |
| 数据请求 | `@tanstack/react-query` |
| 表单 | `react-hook-form` |
| 动画 | `react-native-reanimated` |
| 手势 | `react-native-gesture-handler` |
| 图片 | `expo-image` |
| 图标 | `@expo/vector-icons` |
| 本地存储 | `@react-native-async-storage/async-storage` |
| 安全存储 | `expo-secure-store` |
| 相机/图库 | `expo-image-picker` |
| 推送通知 | `expo-notifications` |
| 位置 | `expo-location` |
| 底部弹窗 | `@gorhom/bottom-sheet` |
| Toast | `react-native-toast-message` |
| UI 组件库 | `react-native-paper` / `NativeWind`（Tailwind for RN） |
