# Flutter Monorepo 实践指南（Melos + Bloc）

> 本文档面向中大型 Flutter 团队，介绍如何用 **Melos** 管理多包仓库、用 **Bloc** 组织状态层，并系统对比「按功能分包」与其他分包策略的优缺点。

---

## 1. 为什么用 Monorepo

Monorepo（单一仓库多包）把多个相互关联的 package 放在同一个 Git 仓库中统一管理。

**适用场景：**

- 一个 App 拆分为多个独立模块（首页、订单、IM、支付……）
- 多端共享代码（C 端 App、B 端 App、内部工具共享同一套领域逻辑）
- 团队规模 5 人以上，需要明确模块边界与责任人

**核心收益：**

| 收益          | 说明                                        |
| ------------- | ------------------------------------------- |
| 原子化提交    | 一次 PR 跨多个包修改，保证一致性            |
| 代码复用      | 公共能力（网络、UI 组件、工具）下沉为独立包 |
| 强制边界      | 包之间通过显式依赖，避免“面条式”耦合        |
| 统一工具链    | 一套 lint、格式化、CI 规则覆盖所有包        |
| 增量构建/测试 | 只对变更影响的包跑测试，加速 CI             |

**代价：**

- 初期搭建成本高（脚手架、CI、依赖治理）
- 仓库体积变大，clone / IDE 索引变慢
- 需要纪律：依赖方向、版本策略必须有人维护

---

## 2. 整体架构概览

典型分层（自上而下依赖）：

```
┌─────────────────────────────────────────────┐
│  apps/            可运行的 App 壳工程          │
│  （组装 features + 配置路由/DI/环境）           │
├─────────────────────────────────────────────┤
│  features/        业务功能包（含 Bloc + UI）    │
│  feature_auth  feature_order  feature_profile  │
├─────────────────────────────────────────────┤
│  packages/        共享基础包                    │
│  core  ui_kit  network  storage  analytics     │
└─────────────────────────────────────────────┘
```

**依赖方向铁律：** `apps → features → packages`，**严禁反向依赖，严禁 feature 之间横向直接依赖**（见第 7 节）。

---

## 3. Melos 包管理

[Melos](https://melos.invertase.dev/) 是 Dart/Flutter 官方生态中最主流的 Monorepo 管理工具，负责多包的 **依赖联动、脚本编排、版本发布**。

### 3.1 安装

```bash
dart pub global activate melos
```

### 3.2 根目录 `melos.yaml`

```yaml
name: my_app_workspace

packages:
  - apps/**
  - features/**
  - packages/**

# 命令默认作用范围
command:
  bootstrap:
    # 使用 pub workspace（Dart 3.5+）可省去 pub get 重复下载
    runPubGetInParallel: true
  version:
    # 遵循约定式提交自动生成 CHANGELOG
    linkToCommits: true
    workspaceChangelog: true

scripts:
  analyze:
    description: 静态分析所有包
    exec: dart analyze .

  test:
    description: 运行所有含测试的包
    run: melos exec --dir-exists="test" -- flutter test
    packageFilters:
      flutter: true

  format:
    exec: dart format . --set-exit-if-changed

  gen:
    description: 跑 build_runner（freezed / json_serializable 等）
    run: melos exec -- dart run build_runner build --delete-conflicting-outputs
    packageFilters:
      dependsOn: build_runner

  # 只对“有改动的包”跑测试，CI 提速关键
  test:diff:
    run: melos exec --diff=origin/main -- flutter test
    packageFilters:
      flutter: true
      dirExists: test
```

> Dart 3.6+ 推荐配合原生 **pub workspaces**（`pubspec.yaml` 里的 `workspace:` 字段），Melos 6 已与之集成，依赖解析更快、版本冲突更少。

### 3.3 常用命令

```bash
melos bootstrap        # 安装所有包依赖并建立本地软链接（核心命令，简写 melos bs）
melos run analyze      # 执行自定义脚本
melos run test:diff    # 只测变更包
melos exec -- <cmd>    # 在每个包目录下执行任意命令
melos version          # 按 conventional commits 自动 bump 版本 + 生成 CHANGELOG
melos publish --dry-run
```

### 3.4 `packageFilters` 过滤器（提效核心）

| 过滤器               | 作用                       |
| -------------------- | -------------------------- |
| `--diff=origin/main` | 仅匹配相对某分支有改动的包 |
| `flutter: true`      | 仅 Flutter 包              |
| `dependsOn: bloc`    | 仅依赖某个包的包           |
| `scope` / `ignore`   | 按包名 glob 匹配/排除      |
| `dirExists: test`    | 存在某目录的包             |

通过 `--diff` 实现「增量 CI」：只对受影响的包跑分析和测试，是大型 Monorepo 的关键提速手段。

---

## 4. Bloc 状态管理在 Monorepo 中的落地

### 4.1 包内分层

每个 feature 包内部建议遵循 **数据 / 领域 / 表现** 三层：

```
feature_order/
├── lib/
│   ├── feature_order.dart        # 唯一 public barrel，对外只暴露必要内容
│   └── src/
│       ├── data/                 # repository 实现、DTO、数据源
│       ├── domain/               # 实体、用例(可选)、repository 抽象接口
│       └── presentation/
│           ├── bloc/             # OrderBloc / Event / State
│           ├── pages/
│           └── widgets/
├── test/
└── pubspec.yaml
```

### 4.2 Bloc 代码示例

```dart
// presentation/bloc/order_event.dart
sealed class OrderEvent {}
class OrderFetched extends OrderEvent {}

// presentation/bloc/order_state.dart  (推荐 freezed)
@freezed
class OrderState with _$OrderState {
  const factory OrderState.initial() = _Initial;
  const factory OrderState.loading() = _Loading;
  const factory OrderState.success(List<Order> orders) = _Success;
  const factory OrderState.failure(String message) = _Failure;
}

// presentation/bloc/order_bloc.dart
class OrderBloc extends Bloc<OrderEvent, OrderState> {
  OrderBloc(this._repo) : super(const OrderState.initial()) {
    on<OrderFetched>(_onFetched);
  }

  final OrderRepository _repo;   // 依赖抽象，不依赖实现

  Future<void> _onFetched(OrderFetched e, Emitter<OrderState> emit) async {
    emit(const OrderState.loading());
    try {
      emit(OrderState.success(await _repo.fetch()));
    } catch (err) {
      emit(OrderState.failure(err.toString()));
    }
  }
}
```

### 4.3 跨包的依赖注入

- **共享 `bloc` / `flutter_bloc` 版本**：在公共 `packages/core` 或根 workspace 统一约束，避免版本分裂导致 `BlocProvider` 类型不匹配。
- **DI 容器**（`get_it` / `injectable` / `provider`）放在 App 壳层组装，feature 包只声明它需要的抽象接口（如 `OrderRepository`），由 App 注入具体实现。
- **跨 feature 通信**：不要让 BlocA 直接 import BlocB。用以下方式解耦：
  - 全局事件总线 / `AppBloc`（放在 core）
  - 路由参数传值
  - 共享领域 service（下沉到 packages）

```dart
// apps/main_app/lib/di.dart
final getIt = GetIt.instance;

void configureDependencies() {
  getIt
    ..registerLazySingleton<OrderRepository>(() => OrderRepositoryImpl(getIt()))
    ..registerFactory(() => OrderBloc(getIt()));
}
```

### 4.4 测试

`bloc_test` 让状态流测试非常简洁，每个 feature 包独立维护自己的测试：

```dart
blocTest<OrderBloc, OrderState>(
  'emits [loading, success] when fetch succeeds',
  build: () => OrderBloc(FakeRepo()),
  act: (b) => b.add(OrderFetched()),
  expect: () => [const OrderState.loading(), isA<_Success>()],
);
```

---

## 5. 分包策略对比（核心）

这是分包架构最关键的决策。下面对比四种主流策略。

### 5.1 策略 A：按功能/特性分包（Feature-based 推荐

每个业务功能是一个包，**包内自带数据层、领域层、表现层（含 Bloc）**，是一个“垂直切片”。

```
features/
├── feature_auth/      （登录、注册、找回密码）
├── feature_order/     （订单列表、详情、退款）
├── feature_profile/   （个人中心、设置）
└── feature_payment/
```

**优点：**

- **高内聚**：一个功能的所有代码集中在一处，改需求只动一个包
- **团队可并行**：一个团队/人负责一个 feature，代码冲突少，责任清晰
- **可插拔**：feature 能整体启用/下线，便于 A/B、灰度、按端裁剪
- **认知负担低**：新人按业务理解代码，符合产品视角
- **增量构建友好**：改一个功能只触发该包及其下游的测试/构建

**缺点：**

- 需要严格的依赖治理，否则容易出现 feature 间隐性耦合
- 跨 feature 复用的代码必须及时下沉到 packages，否则重复
- 初期需要定义清楚“什么算一个 feature”的粒度，划分不当会过碎或过粗

### 5.2 策略 B：按技术层/类型分包（Layer-based）

按技术职责横向切分，所有功能的同类代码放一起。

```
packages/
├── data/          所有功能的 repository、API、model
├── domain/        所有功能的实体、用例
├── presentation/  所有功能的页面、Bloc
└── core/
```

**优点：**

- 层次边界清晰，强制贯彻 Clean Architecture
- 对“层”的技术规范容易统一（如所有 data 层统一用某 ORM）

**缺点：**

- **低内聚**：改一个订单需求要同时改 data/domain/presentation 三个包，跳来跳去
- **难以并行**：所有人都在改同样的几个大包，冲突频繁
- **包体积膨胀**：每层都是巨型包，增量构建/测试收益小
- 不利于 feature 级别的裁剪与灰度

> 适用：领域逻辑高度统一、功能数量少、强调架构纪律的中小项目。功能一多就会退化成「大泥球」。

### 5.3 策略 C：混合分包（Feature + Layer） 实战最常用

**外层按 feature 分包，feature 包内部再按 layer 分目录（不是分包）。** 即第 4.1 节的结构。

```
features/feature_order/lib/src/
├── data/
├── domain/
└── presentation/
```

**优点：**

- 兼得 feature 的高内聚 + layer 的清晰分层
- 包数量可控（按 feature，不会爆炸）
- 既能并行开发，又有架构纪律

**缺点：**

- 需要团队对“目录级分层”有共识，靠约定而非编译器强制
- 若某层确实需要跨 feature 强复用，仍需下沉到 packages

> 这是大多数成熟 Flutter Monorepo 的实际选择：**feature 分包 + 包内分层目录**。

### 5.4 策略 D：单包 + 目录分文件夹（不分包）

整个 App 一个 package，靠文件夹组织（很多脚手架的默认形态）。

**优点：**

- 上手最快，无需 Melos / 依赖管理
- 小项目、原型、个人项目足够用

**缺点：**

- **无编译期边界**：任何文件能 import 任何文件，耦合无法约束
- 全量构建，无增量收益
- 团队协作冲突多，难以分工
- 无法做模块级复用与裁剪

### 5.5 总览对比表

| 维度         |   A 按功能   | B 按技术层 |       C 混合       | D 单包目录 |
| ------------ | :----------: | :--------: | :----------------: | :--------: |
| 内聚性       |      高      |     低     |         高         |     中     |
| 团队并行     |      强      |     弱     |         强         |     弱     |
| 编译期边界   |      强      |     强     |         强         |     无     |
| 增量构建收益 |      高      |     低     |         高         |     无     |
| 架构纪律     | 中（需治理） |     强     |         强         |     弱     |
| 包数量       |      多      |     少     |        适中        |     1      |
| 上手成本     |      中      |     中     |         中         |     低     |
| 可裁剪/灰度  |      强      |     弱     |         强         |     无     |
| 适用规模     |    中大型    |   中小型   | **中大型（推荐）** | 小型/原型  |

**结论：**

- 中大型团队 / 多端 → **策略 C（feature 分包 + 包内 layer 分目录）**
- 功能少、强架构纪律 → 策略 B 可接受
- 原型 / 个人项目 → 策略 D 起步，长大后迁移到 C

---

## 6. 推荐目录结构

```
my_app_workspace/
├── melos.yaml
├── pubspec.yaml                 # pub workspace 根
├── analysis_options.yaml        # 全局 lint 规则
│
├── apps/
│   ├── customer_app/            # C 端壳工程
│   │   ├── lib/
│   │   │   ├── main.dart
│   │   │   ├── di.dart          # 依赖注入组装
│   │   │   ├── router.dart      # 路由聚合各 feature
│   │   │   └── app.dart
│   │   └── pubspec.yaml
│   └── merchant_app/            # B 端壳工程（复用部分 feature/packages）
│
├── features/
│   ├── feature_auth/
│   ├── feature_order/
│   ├── feature_profile/
│   └── feature_payment/
│
└── packages/
    ├── core/                    # 常量、扩展、Result、failure、基础 Bloc 工具
    ├── ui_kit/                  # 设计系统：主题、组件、tokens
    ├── network/                 # Dio 封装、拦截器、错误映射
    ├── storage/                 # 本地存储抽象（kv / db）
    ├── analytics/               # 埋点抽象
    └── l10n/                    # 国际化
```

每个 feature 的 `pubspec.yaml`：

```yaml
name: feature_order
publish_to: none

dependencies:
  flutter:
    sdk: flutter
  flutter_bloc: ^9.0.0
  freezed_annotation: ^2.4.0
  # 只依赖共享基础包，不依赖其他 feature
  core:
  ui_kit:
  network:

dev_dependencies:
  bloc_test: ^10.0.0
  build_runner: ^2.4.0
  freezed: ^2.5.0
  mocktail: ^1.0.0
```

---

## 7. 依赖规则与边界约束

边界靠工具强制，而非靠口头约定。

### 7.1 依赖方向规则

```
apps  ────────►  features  ────────►  packages
  │                                       ▲
  └───────────────────────────────────────┘
        （apps 可直接用 packages）

 apps    依赖 features、packages
 features 依赖 packages
 features 依赖 features          ← 横向依赖禁止
 packages 依赖 features / apps   ← 反向依赖禁止
```

### 7.2 用 `import_lint` / 自定义 lint 强制边界

可借助 `custom_lint` + 自定义规则，或在 CI 加脚本检测“feature 互相 import”。简易做法：

```bash
# CI 中检测 feature 横向依赖
melos exec --scope="feature_*" -- \
  "grep -rL 'package:feature_' lib/ || exit 0"
```

### 7.3 公共代码下沉原则

> 当某段逻辑被 **2 个及以上 feature** 需要时，下沉到 `packages/`。

- 两个 feature 都要的 UI 组件 → `ui_kit`
- 两个 feature 都要的领域模型 → 新建 `packages/domain_xxx` 或放 `core`
- 临时一处用 → 留在 feature 内，不要过早抽象

---

## 8. CI/CD 与版本管理

### 8.1 增量 CI（关键）

```yaml
# GitHub Actions 片段
- run: dart pub global activate melos
- run: melos bootstrap
- run: melos run analyze
- run: melos exec --diff=origin/${{ github.base_ref }} -- flutter test
```

只对相对目标分支有改动的包跑测试，大幅缩短流水线。

### 8.2 版本策略

| 策略                        | 说明                 | 适用                      |
| --------------------------- | -------------------- | ------------------------- |
| **统一版本（Fixed）**       | 所有包共用一个版本号 | 内部 App，不对外发包      |
| **独立版本（Independent）** | 各包独立 semver      | 对外发布 SDK / 多团队复用 |

Melos 配合 **Conventional Commits**（`feat:` / `fix:` / `BREAKING CHANGE:`）自动 bump 版本并生成 CHANGELOG：

```bash
melos version --no-private   # 自动按提交记录计算版本
melos publish                # 发布到 pub.dev / 私有源
```

### 8.3 代码生成联动

```bash
melos run gen   # 一键对所有用到 freezed/json_serializable 的包跑 build_runner
```

---

## 9. 常见问题与最佳实践

**Q：feature 之间确实需要跳转/通信怎么办？**
A：通过 App 层的路由解耦（feature 只声明路由名与参数契约），或把共享状态/服务下沉到 `core`。绝不让 feature 直接 import 另一个 feature。

**Q：Bloc 版本如何统一？**
A：在 pub workspace 根的依赖约束里锁定 `bloc` / `flutter_bloc` 版本，所有包共用，避免类型不一致。

**Q：feature 粒度怎么定？**
A：以“一个独立可上线/可灰度的用户场景”为单位。太细（一个页面一个包）会管理爆炸，太粗（整个业务一个包）失去内聚价值。

**Q：每个包都要 barrel 文件吗？**
A：是。每个包只通过 `lib/<package>.dart` 暴露公共 API，内部实现放 `src/`，外部无法 import `src/` 下文件（Dart 的 `src` 约定 + lint 强制）。

**Q：IDE 变慢怎么办？**
A：用 pub workspace 减少重复依赖解析；必要时按需打开子目录而非整个 workspace。

### 最佳实践清单

- 采用 **策略 C**：feature 分包 + 包内 layer 分目录
- 严守依赖方向 `apps → features → packages`，CI 强制
- feature 间零横向依赖，共享逻辑一律下沉
- Bloc + freezed + bloc_test，state 用 sealed/freezed 建模
- DI 在 App 层组装，feature 只依赖抽象
- 用 `melos exec --diff` 做增量 lint/test
- 公共依赖版本在 workspace 根统一锁定
- 每包一个 barrel，实现藏在 `src/`

---

## 参考资料

- Melos 官方文档：https://melos.invertase.dev/
- Bloc 官方文档：https://bloclibrary.dev/
- Dart Pub Workspaces：https://dart.dev/tools/pub/workspaces
- Very Good Ventures ：https://engineering.verygood.ventures/general-practices/philosophy/
