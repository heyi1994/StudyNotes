# Godot Layout

## 1. 基础概念

### Godot UI 的两大节点体系

| 体系        | 根类      | 用途               |
| ----------- | --------- | ------------------ |
| 2D 游戏对象 | `Node2D`  | 精灵、角色、地图等 |
| UI 界面元素 | `Control` | 按钮、标签、面板等 |

> **重要：** UI 布局只涉及 `Control` 及其子类。不要把 `Node2D` 和 `Control` 混用做 UI。

### 坐标系

- 原点 `(0, 0)` 在屏幕**左上角**
- X 轴向右为正，Y 轴向下为正
- Control 节点的位置由 **Anchor + Offset** 共同决定，而不是简单的 `position`

---

## 2. Control 节点

所有 UI 元素的基类，提供布局、输入、主题等核心功能。

### 关键属性

```
Control
├── Anchor（锚点）    — 相对父节点的基准位置（0.0 ~ 1.0）
├── Offset（偏移）    — 相对锚点的像素偏移
├── Size              — 节点的宽高
├── Min Size          — 最小尺寸（容器布局中生效）
├── Grow Direction    — 尺寸增长方向
├── Mouse Filter      — 鼠标事件处理方式
└── Focus Mode        — 键盘焦点模式
```

### 常用 Control 子类

| 节点                    | 说明       |
| ----------------------- | ---------- |
| `Label`                 | 显示文本   |
| `Button`                | 可点击按钮 |
| `TextEdit` / `LineEdit` | 文本输入   |
| `Panel`                 | 背景面板   |
| `TextureRect`           | 显示图片   |
| `ProgressBar`           | 进度条     |
| `Slider`                | 滑动条     |
| `ScrollContainer`       | 可滚动区域 |
| `TabContainer`          | 标签页     |
| `Tree`                  | 树形列表   |

---

## 3. 锚点 Anchor

锚点定义了 Control 节点**相对于父节点的基准位置**，取值范围 `0.0 ~ 1.0`。

### 四个锚点值

```
anchor_left   — 左边缘相对父节点宽度的比例
anchor_top    — 上边缘相对父节点高度的比例
anchor_right  — 右边缘相对父节点宽度的比例
anchor_bottom — 下边缘相对父节点高度的比例
```

### 可视化理解

```
父节点区域：
┌─────────────────────┐
│ (0,0)         (1,0) │
│                     │
│                     │
│ (0,1)         (1,1) │
└─────────────────────┘
```

### Godot 内置 Anchor 预设

在编辑器顶部 Layout 菜单可以快速选择：

| 预设名        | anchor 值 (L, T, R, B) | 效果           |
| ------------- | ---------------------- | -------------- |
| Top Left      | 0, 0, 0, 0             | 固定在左上角   |
| Top Right     | 1, 0, 1, 0             | 固定在右上角   |
| Bottom Left   | 0, 1, 0, 1             | 固定在左下角   |
| Bottom Right  | 1, 1, 1, 1             | 固定在右下角   |
| Center        | 0.5, 0.5, 0.5, 0.5     | 固定在中心     |
| Left Wide     | 0, 0, 0, 1             | 左侧全高条     |
| Right Wide    | 1, 0, 1, 1             | 右侧全高条     |
| Top Wide      | 0, 0, 1, 0             | 顶部全宽条     |
| Bottom Wide   | 0, 1, 1, 1             | 底部全宽条     |
| **Full Rect** | 0, 0, 1, 1             | **撑满父节点** |

### GDScript 设置锚点

```gdscript
# 方式一：逐个设置
$Panel.anchor_left   = 0.0
$Panel.anchor_top    = 0.0
$Panel.anchor_right  = 1.0
$Panel.anchor_bottom = 1.0

# 方式二：使用预设（推荐）
$Panel.set_anchors_preset(Control.PRESET_FULL_RECT)

# 常用预设常量
Control.PRESET_TOP_LEFT        # 左上
Control.PRESET_TOP_RIGHT       # 右上
Control.PRESET_BOTTOM_LEFT     # 左下
Control.PRESET_BOTTOM_RIGHT    # 右下
Control.PRESET_CENTER          # 居中
Control.PRESET_CENTER_TOP      # 顶部居中
Control.PRESET_CENTER_BOTTOM   # 底部居中
Control.PRESET_LEFT_WIDE       # 左侧全高
Control.PRESET_RIGHT_WIDE      # 右侧全高
Control.PRESET_TOP_WIDE        # 顶部全宽
Control.PRESET_BOTTOM_WIDE     # 底部全宽
Control.PRESET_FULL_RECT       # 撑满
```

---

## 4. 边距 Offset / Margin

Offset 是相对于锚点位置的**像素偏移**，决定节点最终的精确位置。

### Godot 4 中的 Offset（Godot 3 叫 Margin）

```
offset_left   — 左边缘距锚点的像素距离
offset_top    — 上边缘距锚点的像素距离
offset_right  — 右边缘距锚点的像素距离
offset_bottom — 下边缘距锚点的像素距离
```

### 计算公式

```
实际左边缘 X = 父节点宽度 × anchor_left + offset_left
实际上边缘 Y = 父节点高度 × anchor_top  + offset_top
实际右边缘 X = 父节点宽度 × anchor_right + offset_right
实际下边缘 Y = 父节点高度 × anchor_bottom + offset_bottom
```

### 示例：右下角固定大小按钮

```gdscript
var btn = $Button

# 锚点设为右下角
btn.set_anchors_preset(Control.PRESET_BOTTOM_RIGHT)

# offset 设置偏移（负值表示向内缩进）
btn.offset_left   = -120  # 按钮左边缘在锚点左侧 120px
btn.offset_top    = -50   # 按钮上边缘在锚点上方 50px
btn.offset_right  = -10   # 按钮右边缘在锚点左侧 10px（距右边10px间距）
btn.offset_bottom = -10   # 按钮下边缘在锚点上方 10px（距底部10px间距）
```

结果：按钮固定在右下角，尺寸 110×40，距边缘 10px。

### 快速设置位置和大小

```gdscript
# 设置位置（会影响 offset）
$Control.position = Vector2(100, 200)

# 设置大小
$Control.size = Vector2(300, 100)

# 同时设置（推荐用 set_position + set_size 或直接编辑器拖）
```

---

## 5. 容器 Container

Container 是自动管理子节点布局的特殊 Control 节点。

### 核心规则

1. Container 会**自动控制子节点的位置和大小**
2. 子节点的 Anchor 和 Offset 在容器内**不生效**
3. 子节点通过 `minimum_size` 和 **Size Flags** 影响布局
4. Container 本身也是 Control，可以嵌套

### Size Flags（尺寸标志）

控制子节点在容器中如何分配空间：

| 标志                 | 值  | 说明                   |
| -------------------- | --- | ---------------------- |
| `SIZE_SHRINK_BEGIN`  | 0   | 缩至最小，靠起始端对齐 |
| `SIZE_FILL`          | 1   | 填充分配到的空间       |
| `SIZE_EXPAND`        | 2   | 争抢剩余空间           |
| `SIZE_EXPAND_FILL`   | 3   | 争抢并填充（最常用）   |
| `SIZE_SHRINK_CENTER` | 4   | 缩至最小，居中对齐     |
| `SIZE_SHRINK_END`    | 8   | 缩至最小，靠末端对齐   |

```gdscript
# 让按钮在 HBoxContainer 中横向撑满
$Button.size_flags_horizontal = Control.SIZE_EXPAND_FILL

# 让标签在 VBoxContainer 中纵向撑满
$Label.size_flags_vertical = Control.SIZE_EXPAND_FILL
```

---

## 6. 常用容器详解

### 6.1 HBoxContainer — 水平排列

子节点从左到右依次排列。

```
┌──────────────────────────────┐
│ [Btn1] [Btn2] [Btn3]         │
└──────────────────────────────┘
```

```gdscript
var hbox = HBoxContainer.new()
hbox.set_anchors_preset(Control.PRESET_FULL_RECT)

var btn1 = Button.new()
btn1.text = "开始"
btn1.size_flags_horizontal = Control.SIZE_EXPAND_FILL

var btn2 = Button.new()
btn2.text = "设置"
btn2.size_flags_horizontal = Control.SIZE_EXPAND_FILL

hbox.add_child(btn1)
hbox.add_child(btn2)
add_child(hbox)
```

**属性：**

- `alignment` — 子节点对齐方式（BEGIN / CENTER / END）
- `separation` — 子节点间距（像素）

---

### 6.2 VBoxContainer — 垂直排列

子节点从上到下依次排列。

```
┌──────────┐
│ [  开始  ] │
│ [  设置  ] │
│ [  退出  ] │
└──────────┘
```

```gdscript
var vbox = VBoxContainer.new()
vbox.set_anchors_preset(Control.PRESET_CENTER)

for text in ["开始游戏", "载入存档", "退出"]:
    var btn = Button.new()
    btn.text = text
    btn.custom_minimum_size = Vector2(200, 50)
    vbox.add_child(btn)
```

---

### 6.3 GridContainer — 网格排列

按固定列数排列子节点。

```
┌────────────────────┐
│ [A] [B] [C]        │
│ [D] [E] [F]        │
│ [G] [H]            │
└────────────────────┘
```

```gdscript
var grid = GridContainer.new()
grid.columns = 3  # 设置列数

for i in range(9):
    var btn = Button.new()
    btn.text = str(i)
    grid.add_child(btn)
```

---

### 6.4 MarginContainer — 外边距容器

为内容添加边距，只能有一个有效子节点。

```gdscript
var margin = MarginContainer.new()

# 设置四边边距（Godot 4 用 Theme Override）
margin.add_theme_constant_override("margin_left",   20)
margin.add_theme_constant_override("margin_top",    20)
margin.add_theme_constant_override("margin_right",  20)
margin.add_theme_constant_override("margin_bottom", 20)

var label = Label.new()
label.text = "带边距的内容"
margin.add_child(label)
```

---

### 6.5 PanelContainer — 面板容器

带背景样式的容器，常用于对话框、卡片等。

```gdscript
var panel = PanelContainer.new()
panel.set_anchors_preset(Control.PRESET_CENTER)
panel.custom_minimum_size = Vector2(300, 200)

var vbox = VBoxContainer.new()
panel.add_child(vbox)

var title = Label.new()
title.text = "提示"
vbox.add_child(title)

var content = Label.new()
content.text = "确定要退出吗？"
vbox.add_child(content)
```

---

### 6.6 ScrollContainer — 滚动容器

内容超出区域时可滚动查看。

```gdscript
var scroll = ScrollContainer.new()
scroll.set_anchors_preset(Control.PRESET_FULL_RECT)

# 必须设置子节点可以超出滚动容器
var vbox = VBoxContainer.new()
vbox.size_flags_horizontal = Control.SIZE_EXPAND_FILL

for i in range(50):
    var label = Label.new()
    label.text = "第 %d 行内容" % i
    vbox.add_child(label)

scroll.add_child(vbox)
```

**属性：**

- `horizontal_scroll_mode` — 水平滚动条显示方式
- `vertical_scroll_mode` — 垂直滚动条显示方式
- 值：`SCROLL_MODE_DISABLED` / `SCROLL_MODE_AUTO` / `SCROLL_MODE_ALWAYS`

---

### 6.7 CenterContainer — 居中容器

将子节点自动居中。

```gdscript
var center = CenterContainer.new()
center.set_anchors_preset(Control.PRESET_FULL_RECT)

var label = Label.new()
label.text = "我在正中央"
center.add_child(label)
```

---

### 6.8 AspectRatioContainer — 等比例容器

保持子节点宽高比。

```gdscript
var arc = AspectRatioContainer.new()
arc.ratio = 16.0 / 9.0
arc.stretch_mode = AspectRatioContainer.STRETCH_FIT
```

**stretch_mode 选项：**

- `STRETCH_WIDTH_CONTROLS_HEIGHT` — 宽度决定高度
- `STRETCH_HEIGHT_CONTROLS_WIDTH` — 高度决定宽度
- `STRETCH_FIT` — 适应（保留空白）
- `STRETCH_COVER` — 覆盖（裁切）

---

### 6.9 TabContainer — 标签页容器

多页面切换。

```gdscript
var tabs = TabContainer.new()
tabs.set_anchors_preset(Control.PRESET_FULL_RECT)

# 每个直接子节点成为一个标签页
# 子节点的 name 属性作为标签标题
var page1 = VBoxContainer.new()
page1.name = "设置"
tabs.add_child(page1)

var page2 = VBoxContainer.new()
page2.name = "音频"
tabs.add_child(page2)
```

---

### 6.10 SplitContainer / HSplitContainer / VSplitContainer

可拖动分割两个区域。

```gdscript
var split = HSplitContainer.new()
split.set_anchors_preset(Control.PRESET_FULL_RECT)
split.split_offset = 200  # 初始分割位置

var left_panel = Panel.new()
var right_panel = Panel.new()

split.add_child(left_panel)
split.add_child(right_panel)
```

---

### 6.11 FlowContainer（Godot 4.1+）

类似 CSS Flexbox 的换行布局。

```gdscript
var flow = HFlowContainer.new()
# 子节点超出宽度时自动换行
```

---

## 7. 主题与样式

### Theme 系统

Godot 使用 `Theme` 资源统一管理 UI 样式。

```gdscript
# 创建主题
var theme = Theme.new()

# 设置字体大小
theme.set_font_size("font_size", "Label", 24)

# 设置颜色
theme.set_color("font_color", "Label", Color.WHITE)

# 应用主题（会影响所有子节点）
$Panel.theme = theme
```

### StyleBox

StyleBox 定义节点的背景/边框样式。

```gdscript
# 纯色背景
var style = StyleBoxFlat.new()
style.bg_color = Color(0.2, 0.2, 0.2, 0.9)
style.border_width_left   = 2
style.border_width_right  = 2
style.border_width_top    = 2
style.border_width_bottom = 2
style.border_color = Color.WHITE
style.corner_radius_top_left     = 8
style.corner_radius_top_right    = 8
style.corner_radius_bottom_left  = 8
style.corner_radius_bottom_right = 8

$Panel.add_theme_stylebox_override("panel", style)
```

### 常用 Theme Override 方法

```gdscript
# 颜色
node.add_theme_color_override("font_color", Color.RED)

# 字体大小
node.add_theme_font_size_override("font_size", 32)

# 常量（间距等）
node.add_theme_constant_override("separation", 10)

# StyleBox
node.add_theme_stylebox_override("normal", style_box)
```

---

## 8. 响应式布局

### 获取视口尺寸

```gdscript
func _ready():
    var viewport_size = get_viewport().get_visible_rect().size
    print("屏幕尺寸：", viewport_size)

func _notification(what):
    if what == NOTIFICATION_RESIZED:
        _on_resize()

func _on_resize():
    var size = get_viewport_rect().size
    # 根据尺寸调整布局
```

### 连接尺寸变化信号

```gdscript
func _ready():
    get_tree().root.size_changed.connect(_on_window_size_changed)

func _on_window_size_changed():
    var new_size = DisplayServer.window_get_size()
    print("窗口变为：", new_size)
```

### 项目设置中的拉伸模式

**Project Settings → Display → Window → Stretch**

| 设置                 | 说明                       |
| -------------------- | -------------------------- |
| `mode: viewport`     | 整个视口缩放，像素风格推荐 |
| `mode: canvas_items` | 各元素独立缩放，UI 推荐    |
| `aspect: keep`       | 保持宽高比，留黑边         |
| `aspect: expand`     | 保持宽高比，扩展视野       |
| `aspect: ignore`     | 拉伸填充，可能变形         |

### GDScript 中动态适配

```gdscript
func adapt_layout(screen_size: Vector2):
    if screen_size.x < 768:
        # 手机竖屏布局
        $HBox.vertical = true  # 改为垂直排列
        $Sidebar.hide()
    else:
        # 桌面宽屏布局
        $HBox.vertical = false
        $Sidebar.show()
```

---

## 9. 常见布局实战

### 9.1 游戏 HUD（头部信息栏）

```
┌─────────────────────────────────┐  ← 全屏 CanvasLayer
│ [♥ 100]        [得分: 9999]  [⚙] │  ← 顶部 HBoxContainer
│                                 │
│          游戏区域                │
│                                 │
│ [技能1] [技能2] [技能3]           │  ← 底部 HBoxContainer
└─────────────────────────────────┘
```

```gdscript
# HUD 场景结构：
# CanvasLayer
#   └── Control (FULL_RECT)
#         ├── HBoxContainer (TOP_WIDE, anchor_bottom=0, offset_bottom=60)
#         │     ├── Label (HP)
#         │     ├── Control (SIZE_EXPAND_FILL, 占位拉伸)
#         │     ├── Label (Score)
#         │     └── Button (设置)
#         └── HBoxContainer (BOTTOM_WIDE, anchor_top=1, offset_top=-80)
#               ├── Button (技能1)
#               ├── Button (技能2)
#               └── Button (技能3)

# 顶部栏
var top_bar = HBoxContainer.new()
top_bar.anchor_left   = 0
top_bar.anchor_top    = 0
top_bar.anchor_right  = 1
top_bar.anchor_bottom = 0
top_bar.offset_bottom = 60

# 底部栏
var bot_bar = HBoxContainer.new()
bot_bar.anchor_left   = 0
bot_bar.anchor_top    = 1
bot_bar.anchor_right  = 1
bot_bar.anchor_bottom = 1
bot_bar.offset_top    = -80
```

---

### 9.2 主菜单

```
┌─────────────────────────────────┐
│                                 │
│           游戏标题               │
│                                 │
│         [ 开始游戏 ]             │
│         [ 继续游戏 ]             │
│         [ 设  置  ]             │
│         [ 退  出  ]             │
│                                 │
└─────────────────────────────────┘
```

```gdscript
# 场景结构：
# Control (FULL_RECT)
#   └── CenterContainer (FULL_RECT)
#         └── VBoxContainer
#               ├── Label (标题)
#               ├── Control (custom_minimum_size.y = 40, 间隔)
#               ├── Button (开始游戏)
#               ├── Button (继续游戏)
#               ├── Button (设置)
#               └── Button (退出)

var center = CenterContainer.new()
center.set_anchors_preset(Control.PRESET_FULL_RECT)

var vbox = VBoxContainer.new()
vbox.add_theme_constant_override("separation", 16)

var title = Label.new()
title.text = "我的游戏"
title.add_theme_font_size_override("font_size", 48)
title.horizontal_alignment = HORIZONTAL_ALIGNMENT_CENTER

vbox.add_child(title)

for btn_text in ["开始游戏", "继续游戏", "设置", "退出"]:
    var btn = Button.new()
    btn.text = btn_text
    btn.custom_minimum_size = Vector2(200, 50)
    vbox.add_child(btn)

center.add_child(vbox)
add_child(center)
```

---

### 9.3 对话框弹窗

```
┌─────────────────────────────────┐  ← 半透明遮罩
│         ┌──────────────┐        │
│         │   提示       │        │  ← PanelContainer
│         │              │        │
│         │ 确定要退出吗？│        │
│         │              │        │
│         │ [取消] [确认] │        │
│         └──────────────┘        │
└─────────────────────────────────┘
```

```gdscript
func show_dialog(message: String):
    # 遮罩层
    var overlay = ColorRect.new()
    overlay.set_anchors_preset(Control.PRESET_FULL_RECT)
    overlay.color = Color(0, 0, 0, 0.5)
    add_child(overlay)

    # 对话框面板
    var center = CenterContainer.new()
    center.set_anchors_preset(Control.PRESET_FULL_RECT)
    overlay.add_child(center)

    var panel = PanelContainer.new()
    panel.custom_minimum_size = Vector2(320, 180)
    center.add_child(panel)

    var vbox = VBoxContainer.new()
    vbox.add_theme_constant_override("separation", 20)
    panel.add_child(vbox)

    var margin = MarginContainer.new()
    for side in ["margin_left","margin_top","margin_right","margin_bottom"]:
        margin.add_theme_constant_override(side, 20)
    vbox.add_child(margin)

    var inner = VBoxContainer.new()
    margin.add_child(inner)

    var msg_label = Label.new()
    msg_label.text = message
    msg_label.horizontal_alignment = HORIZONTAL_ALIGNMENT_CENTER
    inner.add_child(msg_label)

    var hbox = HBoxContainer.new()
    hbox.alignment = BoxContainer.ALIGNMENT_CENTER
    hbox.add_theme_constant_override("separation", 20)
    inner.add_child(hbox)

    var cancel_btn = Button.new()
    cancel_btn.text = "取消"
    cancel_btn.custom_minimum_size = Vector2(100, 40)
    cancel_btn.pressed.connect(func(): overlay.queue_free())
    hbox.add_child(cancel_btn)

    var confirm_btn = Button.new()
    confirm_btn.text = "确认"
    confirm_btn.custom_minimum_size = Vector2(100, 40)
    confirm_btn.pressed.connect(func():
        overlay.queue_free()
        get_tree().quit()
    )
    hbox.add_child(confirm_btn)
```

---

### 9.4 背包格子布局

```
┌────────────────────────┐
│ [物品][物品][物品][物品] │
│ [物品][物品][    ][    ] │
│ [    ][    ][    ][    ] │
└────────────────────────┘
```

```gdscript
const SLOT_SIZE = Vector2(64, 64)
const COLUMNS = 4
const ROWS = 3

var grid = GridContainer.new()
grid.columns = COLUMNS

for i in range(COLUMNS * ROWS):
    var slot = Panel.new()
    slot.custom_minimum_size = SLOT_SIZE

    # 添加物品图标（如果有的话）
    if i < inventory.size():
        var icon = TextureRect.new()
        icon.texture = inventory[i].icon
        icon.set_anchors_preset(Control.PRESET_FULL_RECT)
        slot.add_child(icon)

    grid.add_child(slot)

add_child(grid)
```

---

## 10. 调试技巧

### 显示布局边界

在编辑器中开启：**Debug → Visible Collision Shapes** 或直接在运行时：

```gdscript
# 给所有 Control 节点画出边框（调试用）
func _draw():
    draw_rect(Rect2(Vector2.ZERO, size), Color.RED, false, 1.0)
```

### 打印节点布局信息

```gdscript
func print_layout_info(node: Control):
    print("=== %s ===" % node.name)
    print("  position: ", node.position)
    print("  size:     ", node.size)
    print("  global:   ", node.global_position)
    print("  anchor:   L=%s T=%s R=%s B=%s" % [
        node.anchor_left, node.anchor_top,
        node.anchor_right, node.anchor_bottom
    ])
    print("  offset:   L=%s T=%s R=%s B=%s" % [
        node.offset_left, node.offset_top,
        node.offset_right, node.offset_bottom
    ])
```

### 常见问题排查

| 现象                       | 可能原因             | 解决方法                        |
| -------------------------- | -------------------- | ------------------------------- |
| 节点不显示                 | size 为 0            | 检查 minimum_size 或父容器设置  |
| 容器内节点位置乱           | 手动设置了 position  | 改用 size_flags 控制            |
| 节点不跟随窗口缩放         | Anchor 设置错误      | 用预设 FULL_RECT 或正确设置锚点 |
| 布局在编辑器正常但运行时错 | \_ready 中修改了布局 | 用 call_deferred 延迟执行       |
| 滚动容器不能滚动           | 子节点没有超出范围   | 给子节点设置 SIZE_EXPAND_FILL   |

```gdscript
# 布局初始化延迟执行技巧
func _ready():
    call_deferred("_setup_layout")

func _setup_layout():
    # 此时所有节点已完成布局计算
    var real_size = $SomeControl.size
    print("真实尺寸：", real_size)
```

---

## 附录：容器选择速查

```
需要水平排列？          → HBoxContainer
需要垂直排列？          → VBoxContainer
需要网格排列？          → GridContainer
需要居中一个元素？      → CenterContainer
需要添加内边距？        → MarginContainer
需要带背景的容器？      → PanelContainer
需要可滚动区域？        → ScrollContainer
需要保持宽高比？        → AspectRatioContainer
需要可拖动分割？        → HSplitContainer / VSplitContainer
需要标签页切换？        → TabContainer
需要自动换行？          → HFlowContainer / VFlowContainer
```

---
