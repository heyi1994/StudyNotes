# Jetpack Compose

Jetpack Compose 是 Android 官方的现代声明式 UI 框架，使用 Kotlin 编写，完全取代传统 XML 布局。与 Flutter 的设计理念高度相似：组件化、单向数据流、状态驱动 UI。

**与 Flutter 的核心对比：**

| | Jetpack Compose | Flutter |
|---|---|---|
| 语言 | Kotlin | Dart |
| 组件单位 | `@Composable` 函数 | `Widget` 类 |
| 状态 | `remember` + `mutableStateOf` | `StatefulWidget` + `setState` |
| 布局 | `Column` / `Row` / `Box` | `Column` / `Row` / `Stack` |
| 列表 | `LazyColumn` / `LazyRow` | `ListView.builder` |
| 导航 | Navigation Compose | Navigator / GoRouter |
| 主题 | MaterialTheme | ThemeData |

---

## 环境配置

### build.gradle（项目级）

```kotlin
// build.gradle.kts（项目级）
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.kotlin.compose) apply false
}
```

### build.gradle（模块级）

```kotlin
// app/build.gradle.kts
android {
    compileSdk = 35

    defaultConfig {
        minSdk = 24
        targetSdk = 35
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_11
        targetCompatibility = JavaVersion.VERSION_11
    }

    kotlinOptions {
        jvmTarget = "11"
    }

    buildFeatures {
        compose = true
    }
}

dependencies {
    val composeBom = platform("androidx.compose:compose-bom:2025.01.00")
    implementation(composeBom)

    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.ui:ui-graphics")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.activity:activity-compose:1.10.0")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.7")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.8.7")
    implementation("androidx.navigation:navigation-compose:2.8.5")

    debugImplementation("androidx.compose.ui:ui-tooling")
    debugImplementation("androidx.compose.ui:ui-test-manifest")
}
```

### Activity 入口

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()  // 沉浸式边缘绘制
        setContent {
            MyAppTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    App()
                }
            }
        }
    }
}
```

---

## 可组合函数（Composable）

```kotlin
// 基本结构：@Composable 注解 + 大写函数名
@Composable
fun Greeting(name: String) {
    Text(text = "你好，$name！")
}

// 预览（仅 IDE 可见，不影响生产代码）
@Preview(showBackground = true, widthDp = 360)
@Composable
fun GreetingPreview() {
    MyAppTheme {
        Greeting("张三")
    }
}
```

**Composable 函数规则：**

- 只能在其他 `@Composable` 函数中调用
- 函数名用大写开头（PascalCase）
- 无返回值（返回类型为 `Unit`）
- 幂等性：相同输入总产生相同 UI
- 不应有副作用（副作用放在 `LaunchedEffect` 等 Effect API 中）

---

## 基础组件

### Text 文本

```kotlin
Text(text = "普通文字")

Text(
    text = "样式丰富的文字",
    fontSize = 18.sp,
    fontWeight = FontWeight.Bold,
    fontStyle = FontStyle.Italic,
    color = MaterialTheme.colorScheme.primary,
    textAlign = TextAlign.Center,
    lineHeight = 24.sp,
    letterSpacing = 0.5.sp,
    maxLines = 2,
    overflow = TextOverflow.Ellipsis,
    textDecoration = TextDecoration.Underline,
)

// 富文本（AnnotatedString）
Text(
    text = buildAnnotatedString {
        append("普通文字，")
        withStyle(SpanStyle(fontWeight = FontWeight.Bold, color = Color.Red)) {
            append("加粗红字，")
        }
        withStyle(SpanStyle(fontSize = 20.sp)) {
            append("大号文字")
        }
    }
)
```

### Image 图片

```kotlin
// 本地资源
Image(
    painter = painterResource(id = R.drawable.ic_logo),
    contentDescription = "应用图标",
    modifier = Modifier.size(64.dp),
    contentScale = ContentScale.Crop,
)

// 网络图片（使用 Coil）
// implementation("io.coil-kt.coil3:coil-compose:3.0.4")
AsyncImage(
    model = ImageRequest.Builder(LocalContext.current)
        .data("https://example.com/photo.jpg")
        .crossfade(true)
        .build(),
    contentDescription = "用户头像",
    modifier = Modifier
        .size(48.dp)
        .clip(CircleShape),
    contentScale = ContentScale.Crop,
    placeholder = painterResource(R.drawable.placeholder),
    error = painterResource(R.drawable.error_image),
)

// 图标
Icon(
    imageVector = Icons.Default.Favorite,
    contentDescription = "收藏",
    tint = MaterialTheme.colorScheme.error,
)
```

### Button 按钮

```kotlin
// 实心按钮
Button(
    onClick = { /* 处理点击 */ },
    enabled = !isLoading,
    colors = ButtonDefaults.buttonColors(
        containerColor = MaterialTheme.colorScheme.primary,
    ),
    shape = RoundedCornerShape(8.dp),
) {
    if (isLoading) {
        CircularProgressIndicator(
            modifier = Modifier.size(16.dp),
            color = Color.White,
            strokeWidth = 2.dp,
        )
        Spacer(Modifier.width(8.dp))
    }
    Text("提交")
}

// 边框按钮
OutlinedButton(onClick = { }) { Text("取消") }

// 文字按钮
TextButton(onClick = { }) { Text("忘记密码") }

// 图标按钮
IconButton(onClick = { }) {
    Icon(Icons.Default.Search, contentDescription = "搜索")
}

// FAB
FloatingActionButton(onClick = { }) {
    Icon(Icons.Default.Add, contentDescription = "新建")
}

// 扩展 FAB
ExtendedFloatingActionButton(
    onClick = { },
    icon = { Icon(Icons.Default.Add, contentDescription = null) },
    text = { Text("新建笔记") },
)
```

### TextField 输入框

```kotlin
var text by remember { mutableStateOf("") }

// 标准输入框
TextField(
    value = text,
    onValueChange = { text = it },
    label = { Text("邮箱") },
    placeholder = { Text("name@example.com") },
    leadingIcon = { Icon(Icons.Default.Email, contentDescription = null) },
    trailingIcon = {
        if (text.isNotEmpty()) {
            IconButton(onClick = { text = "" }) {
                Icon(Icons.Default.Clear, contentDescription = "清空")
            }
        }
    },
    keyboardOptions = KeyboardOptions(
        keyboardType = KeyboardType.Email,
        imeAction = ImeAction.Next,
    ),
    keyboardActions = KeyboardActions(
        onNext = { focusManager.moveFocus(FocusDirection.Down) }
    ),
    singleLine = true,
    isError = text.isNotEmpty() && !text.contains("@"),
    supportingText = {
        if (text.isNotEmpty() && !text.contains("@")) {
            Text("邮箱格式不正确", color = MaterialTheme.colorScheme.error)
        }
    },
    modifier = Modifier.fillMaxWidth(),
)

// 描边样式
OutlinedTextField(
    value = text,
    onValueChange = { text = it },
    label = { Text("用户名") },
    modifier = Modifier.fillMaxWidth(),
)

// 密码输入框
var passwordVisible by remember { mutableStateOf(false) }
OutlinedTextField(
    value = password,
    onValueChange = { password = it },
    label = { Text("密码") },
    visualTransformation = if (passwordVisible) VisualTransformation.None
                           else PasswordVisualTransformation(),
    trailingIcon = {
        IconButton(onClick = { passwordVisible = !passwordVisible }) {
            Icon(
                if (passwordVisible) Icons.Default.Visibility else Icons.Default.VisibilityOff,
                contentDescription = if (passwordVisible) "隐藏密码" else "显示密码",
            )
        }
    },
)
```

---

## 布局组件

### Column / Row / Box

```kotlin
// Column：垂直排列（类似 Flutter Column）
Column(
    modifier = Modifier
        .fillMaxWidth()
        .padding(16.dp),
    verticalArrangement = Arrangement.spacedBy(12.dp),  // 子项间距
    horizontalAlignment = Alignment.CenterHorizontally,
) {
    Text("标题")
    Text("副标题")
    Button(onClick = {}) { Text("按钮") }
}

// Row：水平排列（类似 Flutter Row）
Row(
    modifier = Modifier.fillMaxWidth(),
    horizontalArrangement = Arrangement.SpaceBetween,
    verticalAlignment = Alignment.CenterVertically,
) {
    Text("左侧文字")
    Icon(Icons.Default.ArrowForward, contentDescription = null)
}

// Box：叠加布局（类似 Flutter Stack）
Box(
    modifier = Modifier.size(120.dp),
    contentAlignment = Alignment.Center,
) {
    Image(
        painter = painterResource(R.drawable.avatar),
        contentDescription = null,
        modifier = Modifier.fillMaxSize().clip(CircleShape),
    )
    // 叠加在图片上的徽章
    Badge(
        modifier = Modifier.align(Alignment.TopEnd)
    ) { Text("3") }
}
```

### Spacer 间距

```kotlin
Column {
    Text("上方")
    Spacer(modifier = Modifier.height(16.dp))
    Text("下方")
}

Row {
    Text("左边")
    Spacer(modifier = Modifier.weight(1f))  // 撑满剩余空间
    Text("右边")
}
```

### Arrangement 与 Alignment

```kotlin
// Arrangement（主轴分布）
Column(verticalArrangement = Arrangement.Top)        // 顶部对齐（默认）
Column(verticalArrangement = Arrangement.Center)     // 居中
Column(verticalArrangement = Arrangement.Bottom)     // 底部
Column(verticalArrangement = Arrangement.SpaceBetween)
Column(verticalArrangement = Arrangement.SpaceAround)
Column(verticalArrangement = Arrangement.SpaceEvenly)
Column(verticalArrangement = Arrangement.spacedBy(8.dp)) // 固定间距

// Alignment（交叉轴对齐）
Column(horizontalAlignment = Alignment.Start)
Column(horizontalAlignment = Alignment.CenterHorizontally)
Column(horizontalAlignment = Alignment.End)

Row(verticalAlignment = Alignment.Top)
Row(verticalAlignment = Alignment.CenterVertically)
Row(verticalAlignment = Alignment.Bottom)
```

---

## Modifier 修饰符

Modifier 是 Compose 布局和样式的核心，**顺序影响结果**：

```kotlin
// 常用 Modifier 链式调用
Modifier
    .fillMaxSize()              // 填充父容器
    .fillMaxWidth()             // 填充宽度
    .fillMaxHeight()
    .width(200.dp)              // 固定宽度
    .height(56.dp)
    .size(48.dp)                // 宽高相等
    .wrapContentSize()          // 包裹内容
    .padding(16.dp)             // 内边距
    .padding(horizontal = 16.dp, vertical = 8.dp)
    .offset(x = 8.dp, y = (-4).dp) // 偏移
    .weight(1f)                 // 在 Row/Column 中按比例分配空间

// 外观
    .background(Color.Blue)
    .background(
        brush = Brush.horizontalGradient(listOf(Color.Blue, Color.Purple))
    )
    .background(MaterialTheme.colorScheme.surface, shape = RoundedCornerShape(12.dp))
    .clip(RoundedCornerShape(12.dp))      // 裁剪形状
    .clip(CircleShape)
    .border(1.dp, Color.Gray, RoundedCornerShape(8.dp))
    .alpha(0.7f)
    .shadow(elevation = 4.dp, shape = RoundedCornerShape(8.dp))

// 交互
    .clickable { /* onClick */ }
    .clickable(
        interactionSource = remember { MutableInteractionSource() },
        indication = ripple(),
    ) { }
    .combinedClickable(onClick = { }, onLongClick = { })
    .toggleable(value = checked, onValueChange = { checked = it })
    .draggable(...)

// 注意：顺序很重要
// padding 在 background 之前：内边距在背景内
Modifier.background(Color.Blue).padding(16.dp)
// background 在 padding 之后：内边距在背景外
Modifier.padding(16.dp).background(Color.Blue)
```

---

## 状态管理

### remember 与 mutableStateOf

```kotlin
@Composable
fun Counter() {
    // remember 在重组时保持状态
    var count by remember { mutableStateOf(0) }

    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        Text("计数：$count", style = MaterialTheme.typography.headlineMedium)
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { count-- }) { Text("-") }
            Button(onClick = { count++ }) { Text("+") }
        }
    }
}

// 复杂状态
var user by remember { mutableStateOf(User("张三", 25)) }

// 不可变更新（类似 Redux）
user = user.copy(age = 26)

// rememberSaveable：Activity 重建（旋转屏幕）后也保持状态
var text by rememberSaveable { mutableStateOf("") }
```

### 状态提升（State Hoisting）

将状态从子组件提升到父组件，保持子组件无状态化（类似 Flutter StatelessWidget）：

```kotlin
// ✗ 状态在组件内部（难以测试和复用）
@Composable
fun CounterInternal() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) { Text("$count") }
}

// ✓ 状态提升到父组件
@Composable
fun Counter(
    count: Int,               // 状态向下传
    onIncrement: () -> Unit,  // 事件向上传
    onDecrement: () -> Unit,
) {
    Row {
        Button(onClick = onDecrement) { Text("-") }
        Text("$count")
        Button(onClick = onIncrement) { Text("+") }
    }
}

// 父组件持有状态
@Composable
fun CounterScreen() {
    var count by remember { mutableStateOf(0) }
    Counter(
        count = count,
        onIncrement = { count++ },
        onDecrement = { count-- },
    )
}
```

---

## ViewModel 与 StateFlow

推荐使用 ViewModel + StateFlow/UiState 管理业务状态：

```kotlin
// UiState 数据类
data class UserUiState(
    val users: List<User> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
    val searchQuery: String = "",
)

// ViewModel
class UserViewModel(
    private val repository: UserRepository,
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    init {
        loadUsers()
    }

    private fun loadUsers() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            try {
                val users = repository.getUsers()
                _uiState.update { it.copy(users = users, isLoading = false) }
            } catch (e: Exception) {
                _uiState.update { it.copy(error = e.message, isLoading = false) }
            }
        }
    }

    fun onSearchQueryChange(query: String) {
        _uiState.update { it.copy(searchQuery = query) }
        viewModelScope.launch {
            val users = repository.searchUsers(query)
            _uiState.update { it.copy(users = users) }
        }
    }

    fun deleteUser(userId: Int) {
        viewModelScope.launch {
            repository.deleteUser(userId)
            _uiState.update { state ->
                state.copy(users = state.users.filter { it.id != userId })
            }
        }
    }

    fun refresh() = loadUsers()
}

// Composable 中使用
@Composable
fun UserScreen(
    viewModel: UserViewModel = viewModel(),
) {
    // collectAsStateWithLifecycle 比 collectAsState 更安全（生命周期感知）
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    UserContent(
        uiState = uiState,
        onSearchChange = viewModel::onSearchQueryChange,
        onDelete = viewModel::deleteUser,
        onRefresh = viewModel::refresh,
    )
}

@Composable
fun UserContent(
    uiState: UserUiState,
    onSearchChange: (String) -> Unit,
    onDelete: (Int) -> Unit,
    onRefresh: () -> Unit,
) {
    when {
        uiState.isLoading -> {
            Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                CircularProgressIndicator()
            }
        }
        uiState.error != null -> {
            ErrorScreen(message = uiState.error, onRetry = onRefresh)
        }
        else -> {
            LazyColumn {
                item {
                    OutlinedTextField(
                        value = uiState.searchQuery,
                        onValueChange = onSearchChange,
                        label = { Text("搜索用户") },
                        modifier = Modifier.fillMaxWidth().padding(16.dp),
                    )
                }
                items(uiState.users, key = { it.id }) { user ->
                    UserItem(user = user, onDelete = { onDelete(user.id) })
                }
            }
        }
    }
}
```

---

## 副作用（Effect APIs）

### LaunchedEffect

在 Composable 中启动协程，`key` 变化时重新执行：

```kotlin
@Composable
fun UserDetail(userId: Int) {
    var user by remember { mutableStateOf<User?>(null) }

    // userId 变化时重新加载
    LaunchedEffect(userId) {
        user = fetchUser(userId)
    }

    // key = Unit 或 true：只执行一次（组件首次显示时）
    LaunchedEffect(Unit) {
        delay(3000)
        showWelcomeDialog()
    }

    user?.let { Text(it.name) } ?: CircularProgressIndicator()
}
```

### rememberCoroutineScope

获取与 Composable 生命周期绑定的协程作用域，用于事件触发的异步操作：

```kotlin
@Composable
fun SubmitButton(onSubmit: suspend () -> Unit) {
    val scope = rememberCoroutineScope()
    var isLoading by remember { mutableStateOf(false) }

    Button(
        onClick = {
            scope.launch {
                isLoading = true
                try { onSubmit() } finally { isLoading = false }
            }
        },
        enabled = !isLoading,
    ) {
        if (isLoading) CircularProgressIndicator(Modifier.size(16.dp))
        else Text("提交")
    }
}
```

### DisposableEffect

执行需要清理的副作用（类似 Flutter dispose）：

```kotlin
@Composable
fun LifecycleObserver(onStart: () -> Unit, onStop: () -> Unit) {
    val lifecycle = LocalLifecycleOwner.current.lifecycle

    DisposableEffect(lifecycle) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_START -> onStart()
                Lifecycle.Event.ON_STOP  -> onStop()
                else -> {}
            }
        }
        lifecycle.addObserver(observer)
        onDispose {                        // 组件离开时清理
            lifecycle.removeObserver(observer)
        }
    }
}
```

### SideEffect

每次重组后执行（用于同步非 Compose 状态）：

```kotlin
@Composable
fun SyncAnalytics(screenName: String) {
    SideEffect {
        // 每次重组后通知 Analytics
        analytics.trackScreen(screenName)
    }
}
```

### produceState

将非 Compose 数据源（Flow、LiveData 等）转换为 Compose State：

```kotlin
@Composable
fun NetworkStatus(): Boolean {
    val networkAvailable by produceState(initialValue = false) {
        connectivityManager.registerCallback { available ->
            value = available
        }
        awaitDispose {
            connectivityManager.unregisterCallback()
        }
    }
    return networkAvailable
}
```

---

## 列表

### LazyColumn / LazyRow

```kotlin
@Composable
fun MessageList(messages: List<Message>) {
    LazyColumn(
        contentPadding = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp),
    ) {
        // 固定头部
        item {
            Text(
                "消息列表",
                style = MaterialTheme.typography.titleLarge,
                modifier = Modifier.padding(bottom = 8.dp),
            )
        }

        // 数据项（key 优化增删动画）
        items(messages, key = { it.id }) { message ->
            MessageCard(message = message)
        }

        // 分组：stickyHeader 粘性标题
        // stickyHeader { DateHeader(date) }

        // 加载更多
        item {
            if (isLoadingMore) {
                Box(Modifier.fillMaxWidth().padding(16.dp), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
        }
    }
}
```

### 下拉刷新

```kotlin
// implementation("androidx.compose.material:material:1.7.6")（包含 pullRefresh）
// 或使用 material3 的 PullToRefreshBox
import androidx.compose.material3.pulltorefresh.PullToRefreshBox

@Composable
fun RefreshableList(
    items: List<Item>,
    isRefreshing: Boolean,
    onRefresh: () -> Unit,
) {
    PullToRefreshBox(
        isRefreshing = isRefreshing,
        onRefresh = onRefresh,
    ) {
        LazyColumn(Modifier.fillMaxSize()) {
            items(items) { ItemRow(it) }
        }
    }
}
```

### LazyGrid

```kotlin
// 固定列数
LazyVerticalGrid(
    columns = GridCells.Fixed(2),
    contentPadding = PaddingValues(8.dp),
    horizontalArrangement = Arrangement.spacedBy(8.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp),
) {
    items(products, key = { it.id }) { product ->
        ProductCard(product)
    }
}

// 自适应列数（最小宽度）
LazyVerticalGrid(
    columns = GridCells.Adaptive(minSize = 180.dp),
) { ... }
```

### Pager（轮播/页面滑动）

```kotlin
// implementation("androidx.compose.foundation:foundation")
import androidx.compose.foundation.pager.HorizontalPager
import androidx.compose.foundation.pager.rememberPagerState

@Composable
fun ImageCarousel(images: List<String>) {
    val pagerState = rememberPagerState { images.size }

    Column {
        HorizontalPager(
            state = pagerState,
            modifier = Modifier
                .fillMaxWidth()
                .height(200.dp),
        ) { page ->
            AsyncImage(model = images[page], contentDescription = null, modifier = Modifier.fillMaxSize(), contentScale = ContentScale.Crop)
        }

        // 指示点
        Row(
            Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.Center,
        ) {
            repeat(images.size) { i ->
                val color = if (pagerState.currentPage == i)
                    MaterialTheme.colorScheme.primary
                else MaterialTheme.colorScheme.onSurface.copy(alpha = 0.3f)
                Box(Modifier.padding(4.dp).size(8.dp).clip(CircleShape).background(color))
            }
        }
    }
}
```

---

## 导航（Navigation Compose）

```kotlin
// implementation("androidx.navigation:navigation-compose:2.8.5")

// 定义路由（类型安全，推荐 Kotlin Serializable）
@Serializable object HomeRoute
@Serializable object ProfileRoute
@Serializable data class UserDetailRoute(val userId: Int)

@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = HomeRoute,
    ) {
        composable<HomeRoute> {
            HomeScreen(
                onNavigateToProfile = { navController.navigate(ProfileRoute) },
                onNavigateToUser = { id -> navController.navigate(UserDetailRoute(id)) },
            )
        }

        composable<ProfileRoute> {
            ProfileScreen(onBack = { navController.popBackStack() })
        }

        composable<UserDetailRoute> { backStackEntry ->
            val route: UserDetailRoute = backStackEntry.toRoute()
            UserDetailScreen(
                userId = route.userId,
                onBack = { navController.popBackStack() },
            )
        }
    }
}
```

### BottomNavigationBar

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()
    val currentBackStack by navController.currentBackStackEntryAsState()
    val currentRoute = currentBackStack?.destination?.route

    val tabs = listOf(
        TabItem(HomeRoute, "首页", Icons.Default.Home),
        TabItem(SearchRoute, "搜索", Icons.Default.Search),
        TabItem(ProfileRoute, "我的", Icons.Default.Person),
    )

    Scaffold(
        bottomBar = {
            NavigationBar {
                tabs.forEach { tab ->
                    NavigationBarItem(
                        icon = { Icon(tab.icon, contentDescription = tab.label) },
                        label = { Text(tab.label) },
                        selected = currentRoute == tab.route::class.qualifiedName,
                        onClick = {
                            navController.navigate(tab.route) {
                                popUpTo(navController.graph.findStartDestination().id) {
                                    saveState = true
                                }
                                launchSingleTop = true
                                restoreState = true
                            }
                        },
                    )
                }
            }
        }
    ) { paddingValues ->
        NavHost(
            navController = navController,
            startDestination = HomeRoute,
            modifier = Modifier.padding(paddingValues),
        ) {
            composable<HomeRoute>    { HomeScreen() }
            composable<SearchRoute>  { SearchScreen() }
            composable<ProfileRoute> { ProfileScreen() }
        }
    }
}
```

---

## Material 3 主题

### 自定义主题

```kotlin
// ui/theme/Color.kt
val Purple80  = Color(0xFFD0BCFF)
val PurpleGrey80 = Color(0xFFCCC2DC)
val Pink80    = Color(0xFFEFB8C8)
val Purple40  = Color(0xFF6650A4)

// ui/theme/Theme.kt
private val LightColorScheme = lightColorScheme(
    primary         = Purple40,
    secondary       = PurpleGrey40,
    tertiary        = Pink40,
    background      = Color(0xFFFFFBFE),
    surface         = Color(0xFFFFFBFE),
    onPrimary       = Color.White,
    onBackground    = Color(0xFF1C1B1F),
    onSurface       = Color(0xFF1C1B1F),
)

private val DarkColorScheme = darkColorScheme(
    primary   = Purple80,
    secondary = PurpleGrey80,
    tertiary  = Pink80,
)

@Composable
fun MyAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,   // Android 12+ 动态取色
    content: @Composable () -> Unit,
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context) else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else      -> LightColorScheme
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography  = AppTypography,
        content     = content,
    )
}
```

### 使用主题

```kotlin
@Composable
fun ThemedCard() {
    val colors = MaterialTheme.colorScheme
    val typography = MaterialTheme.typography
    val shapes = MaterialTheme.shapes

    Card(
        shape = shapes.large,
        colors = CardDefaults.cardColors(containerColor = colors.surfaceVariant),
    ) {
        Column(Modifier.padding(16.dp)) {
            Text("标题", style = typography.titleLarge, color = colors.onSurfaceVariant)
            Text("副标题", style = typography.bodyMedium, color = colors.onSurfaceVariant.copy(alpha = 0.7f))
        }
    }
}
```

---

## 动画

### 简单动画

```kotlin
// animateFloatAsState：数值平滑过渡
val alpha by animateFloatAsState(
    targetValue = if (visible) 1f else 0f,
    animationSpec = tween(durationMillis = 300),
    label = "alpha",
)

// animateColorAsState：颜色过渡
val backgroundColor by animateColorAsState(
    targetValue = if (isSelected) MaterialTheme.colorScheme.primaryContainer else Color.Transparent,
    label = "bgColor",
)

// animateDpAsState：尺寸过渡
val size by animateDpAsState(
    targetValue = if (expanded) 200.dp else 80.dp,
    animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy),
    label = "size",
)

Box(
    Modifier
        .size(size)
        .background(backgroundColor)
        .alpha(alpha)
)
```

### AnimatedVisibility

```kotlin
AnimatedVisibility(
    visible = isVisible,
    enter = fadeIn() + slideInVertically { -it },
    exit  = fadeOut() + slideOutVertically { -it },
) {
    Card { Text("可见/隐藏动画") }
}
```

### Crossfade

```kotlin
Crossfade(targetState = currentTab, label = "tab") { tab ->
    when (tab) {
        Tab.HOME    -> HomeContent()
        Tab.PROFILE -> ProfileContent()
    }
}
```

### updateTransition（多属性联动）

```kotlin
val transition = updateTransition(targetState = isExpanded, label = "expand")

val height by transition.animateDp(label = "height") { expanded ->
    if (expanded) 200.dp else 80.dp
}
val alpha by transition.animateFloat(label = "alpha") { expanded ->
    if (expanded) 1f else 0.5f
}

Box(Modifier.height(height).alpha(alpha)) { ... }
```

---

## Scaffold（页面脚手架）

```kotlin
@Composable
fun DetailScreen(onBack: () -> Unit) {
    val snackbarHostState = remember { SnackbarHostState() }
    val scope = rememberCoroutineScope()

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("详情") },
                navigationIcon = {
                    IconButton(onClick = onBack) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBack, contentDescription = "返回")
                    }
                },
                actions = {
                    IconButton(onClick = { /* 分享 */ }) {
                        Icon(Icons.Default.Share, contentDescription = "分享")
                    }
                    IconButton(onClick = { /* 更多 */ }) {
                        Icon(Icons.Default.MoreVert, contentDescription = "更多")
                    }
                },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer,
                ),
            )
        },
        floatingActionButton = {
            FloatingActionButton(onClick = { /* 新建 */ }) {
                Icon(Icons.Default.Add, contentDescription = "新建")
            }
        },
        snackbarHost = { SnackbarHost(snackbarHostState) },
    ) { paddingValues ->
        LazyColumn(contentPadding = paddingValues) {
            // 内容
        }

        // 触发 Snackbar
        Button(onClick = {
            scope.launch {
                val result = snackbarHostState.showSnackbar(
                    message = "已删除",
                    actionLabel = "撤销",
                    duration = SnackbarDuration.Short,
                )
                if (result == SnackbarResult.ActionPerformed) {
                    undoDelete()
                }
            }
        }) { Text("删除") }
    }
}
```

---

## 自定义组件

```kotlin
// 带徽章的头像组件
@Composable
fun BadgedAvatar(
    imageUrl: String,
    badgeCount: Int,
    modifier: Modifier = Modifier,
    size: Dp = 48.dp,
    onClick: (() -> Unit)? = null,
) {
    Box(modifier = modifier.size(size)) {
        AsyncImage(
            model = imageUrl,
            contentDescription = null,
            modifier = Modifier
                .fillMaxSize()
                .clip(CircleShape)
                .then(if (onClick != null) Modifier.clickable { onClick() } else Modifier),
            contentScale = ContentScale.Crop,
        )
        if (badgeCount > 0) {
            Badge(
                modifier = Modifier.align(Alignment.TopEnd),
                containerColor = MaterialTheme.colorScheme.error,
            ) {
                Text(
                    text = if (badgeCount > 99) "99+" else badgeCount.toString(),
                    style = MaterialTheme.typography.labelSmall,
                )
            }
        }
    }
}

// 使用
BadgedAvatar(
    imageUrl = user.avatar,
    badgeCount = unreadCount,
    size = 56.dp,
    onClick = { navController.navigate(ProfileRoute) },
)
```

---

## 性能优化

### key 参数

```kotlin
// 为 items 指定 key，优化增删重排时的重组
LazyColumn {
    items(list, key = { it.id }) { item ->
        ItemRow(item)
    }
}
```

### remember 避免重复计算

```kotlin
@Composable
fun ExpensiveComponent(data: List<Int>) {
    // 没有 remember：每次重组都重新计算
    // val sorted = data.sortedDescending()

    // 有 remember：data 变化时才重新计算
    val sorted = remember(data) { data.sortedDescending() }

    LazyColumn {
        items(sorted) { Text("$it") }
    }
}
```

### derivedStateOf

当计算结果依赖多个状态但不需要每次都更新时使用：

```kotlin
val listState = rememberLazyListState()

// 只有从 false 变为 true（或反之）时才触发重组
val showScrollToTop by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 0 }
}

if (showScrollToTop) {
    ScrollToTopButton(onClick = { scope.launch { listState.animateScrollToItem(0) } })
}
```

### 稳定性（Stability）

```kotlin
// ✗ 不稳定类（导致不必要重组）
data class User(val id: Int, val name: String, val tags: MutableList<String>)

// ✓ 稳定类（所有属性不可变）
@Immutable
data class User(val id: Int, val name: String, val tags: List<String>)

// ✓ 或使用 @Stable
@Stable
class UserState(initialUser: User) {
    var user by mutableStateOf(initialUser)
}
```

---

## 常用组件速查

| 组件 | 说明 |
|------|------|
| `Text` | 文本显示 |
| `TextField` / `OutlinedTextField` | 输入框 |
| `Button` / `OutlinedButton` / `TextButton` | 各类按钮 |
| `IconButton` / `FloatingActionButton` | 图标按钮 / FAB |
| `Image` / `AsyncImage`(Coil) | 本地/网络图片 |
| `Icon` | 图标 |
| `Column` / `Row` / `Box` | 基础布局 |
| `Spacer` | 间距 |
| `Scaffold` | 页面脚手架（含 TopBar/FAB/SnackBar） |
| `TopAppBar` / `CenterAlignedTopAppBar` | 顶部导航栏 |
| `NavigationBar` / `NavigationBarItem` | 底部导航栏 |
| `NavigationDrawer` | 侧边抽屉 |
| `Card` | 卡片 |
| `Surface` | 带背景/阴影的容器 |
| `Divider` / `HorizontalDivider` | 分隔线 |
| `LazyColumn` / `LazyRow` | 垂直/水平列表 |
| `LazyVerticalGrid` | 网格列表 |
| `HorizontalPager` / `VerticalPager` | 页面轮播 |
| `Checkbox` / `RadioButton` / `Switch` | 选择控件 |
| `Slider` | 滑块 |
| `CircularProgressIndicator` / `LinearProgressIndicator` | 进度指示器 |
| `AlertDialog` | 弹窗 |
| `ModalBottomSheet` | 底部弹窗 |
| `DropdownMenu` | 下拉菜单 |
| `Chip` / `FilterChip` / `AssistChip` | 标签芯片 |
| `Badge` | 徽章 |
| `SnackbarHost` | 轻提示 |
| `AnimatedVisibility` | 显隐动画 |
| `Crossfade` | 切换动画 |
