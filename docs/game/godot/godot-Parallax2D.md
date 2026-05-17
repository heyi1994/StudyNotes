# Godot 4.3+ Parallax2D 完全学习文档

> 适用版本：Godot 4.3 及以上  
> 节点类型：2D 节点  
> 继承链：`Node2D → Parallax2D`


## 1. 概述

`Parallax2D` 是 Godot 4.3 中引入的新节点，用于实现 **2D 视差滚动**效果。

视差滚动（Parallax Scrolling）是一种通过让不同距离的图层以不同速度移动，来模拟场景纵深感的技术，广泛用于：

- 横版卷轴游戏背景
- 菜单界面动态背景
- 过场动画
- 无限跑酷游戏场景

`Parallax2D` 将原来需要 `ParallaxBackground + ParallaxLayer` 两个节点才能完成的工作，合并进一个节点，更简洁、更灵活。

**核心原理：**

```
相机移动时，Parallax2D 根据 scroll_scale 的比例，
以不同于相机的速度移动自身，从而产生视差错觉。

实际移动量 = 相机移动量 × scroll_scale
```

---

## 2. 与旧节点的对比

### 2.1 节点结构对比

**旧方式（Godot 4.2 及以前）**

```
Node2D
└── ParallaxBackground          ← 管理整体滚动偏移
    ├── ParallaxLayer           ← 控制单层视差参数
    │   └── Sprite2D            ← 实际图像
    └── ParallaxLayer
        └── Sprite2D
```

**新方式（Godot 4.3+）**

```
Node2D
├── Parallax2D                  ← 合二为一
│   └── Sprite2D
└── Parallax2D
    └── Sprite2D
```

### 2.2 属性对比表

| 旧属性 | 新属性 | 说明 |
|--------|--------|------|
| `ParallaxLayer.motion_scale` | `Parallax2D.scroll_scale` | 视差速度比例 |
| `ParallaxLayer.motion_offset` | `Parallax2D.scroll_offset` | 偏移量 |
| `ParallaxLayer.motion_mirroring` | `Parallax2D.repeat_size` | 无缝循环尺寸 |
| `ParallaxBackground.scroll_offset` | 不需要 | 直接操作各层节点 |
| `ParallaxBackground.scroll_base_offset` | 不需要 | 已内置处理 |
| 无 | `Parallax2D.autoscroll` | 自动滚动（新增） |
| 无 | `Parallax2D.screen_offset` | 屏幕偏移（新增） |

### 2.3 优势总结

- 节点数量减半，场景树更清晰
- 新增 `autoscroll` 属性，无需代码即可实现自动滚动
- 支持更精细的范围限制（`limit_begin` / `limit_end`）
- 不再依赖 `ParallaxBackground` 作为父节点，可自由放置于场景树

---

## 3. 节点属性详解

### 3.1 scroll_scale

```
类型：Vector2
默认值：Vector2(1, 1)
```

控制该图层相对于相机移动的速度比例。

| 值 | 效果 |
|----|------|
| `Vector2(0, 0)` | 完全固定，不随相机移动（适合固定 HUD 背景） |
| `Vector2(0.2, 0.2)` | 缓慢移动，产生"远景"效果 |
| `Vector2(0.5, 0.5)` | 以相机一半速度移动 |
| `Vector2(1, 1)` | 完全跟随相机，无视差感 |
| `Vector2(1.5, 1)` | 比相机移动更快（近景强调效果） |

**X 和 Y 可以独立设置：**

```gdscript
# 只在水平方向有视差，垂直方向固定
$BackgroundLayer.scroll_scale = Vector2(0.3, 0.0)
```

---

### 3.2 scroll_offset

```
类型：Vector2
默认值：Vector2(0, 0)
```

在视差滚动基础上额外添加的偏移量，常用于：
- 初始化图层位置
- 动画中手动推动图层

```gdscript
# 将图层向左偏移 100 像素
$Layer.scroll_offset = Vector2(-100, 0)
```

---

### 3.3 repeat_size

```
类型：Vector2
默认值：Vector2(0, 0)
```

设置图层无缝循环的重复尺寸。当相机移动超出该范围时，图层内容自动"重置"，实现无限背景效果。

- `Vector2(0, 0)`：不循环
- `Vector2(1024, 0)`：水平方向每 1024px 循环一次，垂直不循环
- `Vector2(512, 512)`：双向循环

**重要：** `repeat_size` 应与子图像的实际尺寸匹配，否则会出现接缝。

```gdscript
# 图片宽度为 1024px，设置水平循环
$BackgroundLayer.repeat_size = Vector2(1024, 0)
```

---

### 3.4 autoscroll

```
类型：Vector2
默认值：Vector2(0, 0)
单位：像素/秒
```

让图层自动滚动，**不依赖相机移动**。适合：
- 流动的云彩
- 滚动的星空
- 循环的水流

```gdscript
# 向左自动滚动，每秒 50 像素
$CloudLayer.autoscroll = Vector2(-50, 0)

# 斜向滚动
$StarLayer.autoscroll = Vector2(-20, -10)
```

---

### 3.5 screen_offset

```
类型：Vector2
默认值：Vector2(0, 0)
```

相对于屏幕（视口）原点的偏移，与 `scroll_offset` 不同，它是基于屏幕坐标而非世界坐标的偏移。

---

### 3.6 limit_begin / limit_end

```
类型：Vector2
默认值：limit_begin = Vector2(-1e+07, -1e+07)，limit_end = Vector2(1e+07, 1e+07)
```

限制图层可以滚动的范围，超出范围后图层停止移动。适合有边界的关卡，防止背景滚出可见区域。

```gdscript
# 限制水平滚动范围为 0 到 5000 像素
$Layer.limit_begin = Vector2(0, -10000000)
$Layer.limit_end = Vector2(5000, 10000000)
```

---

### 3.7 follow_viewport

```
类型：bool
默认值：true
```

是否跟随当前视口（相机）。设为 `false` 后，视差效果失效，节点表现为普通 Node2D。

---

## 4. 基础使用：搭建视差背景

### 4.1 场景结构

```
Game (Node2D)
├── Parallax2D  [sky]      scroll_scale=(0.05, 0.05)
│   └── Sprite2D           texture=sky.png
├── Parallax2D  [mountain] scroll_scale=(0.15, 0.1)
│   └── Sprite2D           texture=mountain.png
├── Parallax2D  [forest]   scroll_scale=(0.35, 0.0)
│   └── Sprite2D           texture=forest.png
├── Parallax2D  [ground]   scroll_scale=(0.7, 0.0)
│   └── Sprite2D           texture=ground.png
├── Player (CharacterBody2D)
└── Camera2D               (跟随Player)
```

### 4.2 Inspector 设置步骤

1. 创建 `Node2D` 作为根节点
2. 添加子节点 `Parallax2D`
3. 在 `Parallax2D` 下添加 `Sprite2D`，赋值纹理
4. 调整 `Parallax2D` 的 `scroll_scale`（越小越"远"）
5. 重复以上步骤创建多个图层
6. 添加 `Camera2D` 并跟随玩家

### 4.3 scroll_scale 参考值

| 图层 | scroll_scale X | 视觉距离 |
|------|---------------|---------|
| 天空/星空 | 0.0 ~ 0.1 | 极远 |
| 远山 | 0.1 ~ 0.2 | 很远 |
| 中景树木 | 0.3 ~ 0.5 | 中等 |
| 近景植被 | 0.6 ~ 0.8 | 较近 |
| 地面装饰 | 0.9 ~ 1.0 | 贴近玩家 |

---

## 5. 无限循环背景

实现无限滚动背景，关键在于 `repeat_size` 的正确设置。

### 5.1 单张图片循环

```
Parallax2D
  scroll_scale = Vector2(0.3, 0.0)
  repeat_size  = Vector2(1024, 0)   ← 与图片等宽
└── Sprite2D
      texture = background.png      ← 宽度 1024px
      centered = false              ← 建议关闭居中
      position = Vector2(0, 0)
```

### 5.2 多张图片拼接循环

若单张图片太窄，可以横向并排放置多张：

```
Parallax2D
  repeat_size = Vector2(2048, 0)
└── Node2D
    ├── Sprite2D  position=(0, 0)      texture=bg_tile.png
    └── Sprite2D  position=(1024, 0)   texture=bg_tile.png
```

### 5.3 双向循环（俯视角地图）

```gdscript
# 适合俯视角或双向滚动地图
$GroundLayer.repeat_size = Vector2(512, 512)
```

---

## 6. 自动滚动效果

### 6.1 纯自动滚动（云彩、星空）

不需要相机移动，图层自行滚动：

```gdscript
extends Node2D

func _ready() -> void:
    # 云彩缓慢向左飘动
    $CloudLayer.autoscroll = Vector2(-30, 0)
    $CloudLayer.repeat_size = Vector2(1024, 0)
    
    # 远处星星更慢
    $StarLayer.autoscroll = Vector2(-8, 0)
    $StarLayer.repeat_size = Vector2(2048, 0)
```

### 6.2 自动滚动 + 相机视差叠加

`autoscroll` 和 `scroll_scale` 可以同时生效，效果叠加：

```gdscript
# 背景既随相机移动产生视差，又自动向左流动
$WaterLayer.scroll_scale = Vector2(0.5, 0.0)
$WaterLayer.autoscroll  = Vector2(-20, 0)
$WaterLayer.repeat_size = Vector2(512, 0)
```

### 6.3 运行时改变滚动速度

```gdscript
# 根据玩家速度动态调整背景滚动感
func _process(delta: float) -> void:
    var player_speed = player.velocity.x
    $CloudLayer.autoscroll.x = -player_speed * 0.1
```

---

## 7. 鼠标视差动效

常用于主菜单、UI 界面，鼠标移动时背景产生视差偏移，增强立体感。

### 7.1 基础鼠标视差

```gdscript
extends Node2D

@onready var layers = [
    $FarLayer,    # scroll_scale 不影响鼠标视差，手动控制
    $MidLayer,
    $NearLayer,
]

# 各层对鼠标的响应强度
var strengths = [5.0, 15.0, 30.0]

func _process(_delta: float) -> void:
    var viewport_size = get_viewport_rect().size
    var mouse_pos = get_viewport().get_mouse_position()
    
    # 将鼠标位置归一化为 -1.0 到 1.0
    var normalized = (mouse_pos / viewport_size) * 2.0 - Vector2.ONE
    
    for i in layers.size():
        layers[i].scroll_offset = normalized * strengths[i]
```

### 7.2 平滑鼠标视差（带缓动）

```gdscript
extends Node2D

@onready var layers = [$FarLayer, $MidLayer, $NearLayer]
var strengths = [8.0, 20.0, 40.0]
var current_offsets: Array[Vector2] = []
var smooth_speed = 3.0

func _ready() -> void:
    current_offsets.resize(layers.size())
    current_offsets.fill(Vector2.ZERO)

func _process(delta: float) -> void:
    var viewport_size = get_viewport_rect().size
    var mouse_pos = get_viewport().get_mouse_position()
    var normalized = (mouse_pos / viewport_size) * 2.0 - Vector2.ONE

    for i in layers.size():
        var target = normalized * strengths[i]
        current_offsets[i] = current_offsets[i].lerp(target, smooth_speed * delta)
        layers[i].scroll_offset = current_offsets[i]
```

### 7.3 陀螺仪视差（移动端）

```gdscript
# 需要开启 Input.use_accumulated_input = false
func _input(event: InputEvent) -> void:
    if event is InputEventJoypadMotion:
        # 使用设备重力/陀螺仪
        pass

# 移动端推荐使用 Input.get_gravity() 或 Input.get_accelerometer()
func _process(delta: float) -> void:
    var gravity = Input.get_gravity()
    var tilt = Vector2(gravity.x, -gravity.y).limit_length(1.0)
    
    for i in layers.size():
        layers[i].scroll_offset = tilt * strengths[i]
```

---

## 8. 滚动范围限制

对于有固定边界的关卡，防止背景滚动过头露出空白区域。

### 8.1 设置水平限制

```gdscript
extends Node2D

# 关卡总宽度为 8192px，视口宽 1280px
const LEVEL_WIDTH = 8192.0
const VIEWPORT_WIDTH = 1280.0

func _ready() -> void:
    for layer in [$FarLayer, $MidLayer, $NearLayer]:
        layer.limit_begin = Vector2(0, -100000)
        layer.limit_end   = Vector2(LEVEL_WIDTH - VIEWPORT_WIDTH, 100000)
```

### 8.2 根据 scroll_scale 计算对应限制

视差图层移动距离比相机短，因此限制值也需要按比例缩小：

```gdscript
func setup_layer_limits(layer: Parallax2D, level_width: float) -> void:
    var scale = layer.scroll_scale.x
    layer.limit_begin = Vector2(0, -100000)
    layer.limit_end   = Vector2(level_width * scale, 100000)
```

---

## 9. 与 AnimationPlayer 结合

适合制作固定的过场动画、开场动画中的视差效果。

### 9.1 场景结构

```
CinematicScene (Node2D)
├── Parallax2D  [sky]
│   └── Sprite2D
├── Parallax2D  [mountain]
│   └── Sprite2D
└── AnimationPlayer
```

### 9.2 在 AnimationPlayer 中设置关键帧

在 AnimationPlayer 中，对各 `Parallax2D` 的 `scroll_offset` 属性制作关键帧动画：

```
轨道：Parallax2D:sky/scroll_offset
  0.0s → Vector2(0, 0)
  3.0s → Vector2(-100, 0)    ← 使用 Ease In/Out 缓动

轨道：Parallax2D:mountain/scroll_offset
  0.0s → Vector2(0, 0)
  3.0s → Vector2(-300, 0)

轨道：Parallax2D:foreground/scroll_offset
  0.0s → Vector2(0, 0)
  3.0s → Vector2(-600, 0)
```

### 9.3 代码触发动画

```gdscript
@onready var anim = $AnimationPlayer

func play_intro() -> void:
    anim.play("parallax_intro")

func _on_intro_finished() -> void:
    # 动画播放完毕，切换到游戏场景
    get_tree().change_scene_to_file("res://scenes/game.tscn")
```

---

## 10. 与 Tween 结合

适合响应游戏事件的动态视差动画（如冲击波、切换关卡）。

### 10.1 冲击波视差抖动

```gdscript
extends Node2D

@onready var layers = [$FarLayer, $MidLayer, $NearLayer]
var shake_strengths = [5.0, 12.0, 25.0]

func shake_parallax(duration: float = 0.5) -> void:
    for i in layers.size():
        var tween = create_tween()
        tween.set_trans(Tween.TRANS_SINE)
        tween.set_ease(Tween.EASE_IN_OUT)
        
        var strength = shake_strengths[i]
        tween.tween_property(layers[i], "scroll_offset",
            Vector2(strength, 0), duration * 0.25)
        tween.tween_property(layers[i], "scroll_offset",
            Vector2(-strength, 0), duration * 0.5)
        tween.tween_property(layers[i], "scroll_offset",
            Vector2.ZERO, duration * 0.25)
```

### 10.2 场景切换时的视差推入

```gdscript
func transition_to_next_level() -> void:
    var tween = create_tween()
    tween.set_parallel(true)
    
    # 各层以不同速度推出屏幕
    tween.tween_property($FarLayer, "scroll_offset",
        Vector2(-200, 0), 0.8).set_trans(Tween.TRANS_CUBIC)
    tween.tween_property($MidLayer, "scroll_offset",
        Vector2(-500, 0), 0.6).set_trans(Tween.TRANS_CUBIC)
    tween.tween_property($NearLayer, "scroll_offset",
        Vector2(-1000, 0), 0.4).set_trans(Tween.TRANS_CUBIC)
    
    await tween.finished
    get_tree().change_scene_to_file("res://scenes/level2.tscn")
```

---

## 11. 代码动态创建 Parallax2D

适合程序化生成场景或从数据驱动背景配置。

### 11.1 基础动态创建

```gdscript
extends Node2D

func create_parallax_layer(
    texture: Texture2D,
    scale: Vector2,
    repeat: Vector2,
    z_index: int = 0
) -> Parallax2D:
    
    var layer = Parallax2D.new()
    layer.scroll_scale = scale
    layer.repeat_size = repeat
    layer.z_index = z_index
    
    var sprite = Sprite2D.new()
    sprite.texture = texture
    sprite.centered = false
    layer.add_child(sprite)
    
    add_child(layer)
    return layer

func _ready() -> void:
    var sky_tex = preload("res://textures/sky.png")
    var mountain_tex = preload("res://textures/mountain.png")
    
    create_parallax_layer(sky_tex,      Vector2(0.05, 0.0), Vector2(1024, 0), -10)
    create_parallax_layer(mountain_tex, Vector2(0.2,  0.0), Vector2(512,  0), -5)
```

### 11.2 从配置数组批量创建

```gdscript
extends Node2D

const LAYER_CONFIG = [
    { "texture": "res://textures/sky.png",
      "scale": Vector2(0.05, 0.0), "repeat": Vector2(2048, 0),
      "scroll": Vector2(-5, 0),    "z": -20 },
    { "texture": "res://textures/clouds.png",
      "scale": Vector2(0.15, 0.0), "repeat": Vector2(1024, 0),
      "scroll": Vector2(-15, 0),   "z": -15 },
    { "texture": "res://textures/mountain.png",
      "scale": Vector2(0.3, 0.0),  "repeat": Vector2(512, 0),
      "scroll": Vector2(0, 0),     "z": -10 },
]

func _ready() -> void:
    for config in LAYER_CONFIG:
        var layer = Parallax2D.new()
        layer.scroll_scale  = config["scale"]
        layer.repeat_size   = config["repeat"]
        layer.autoscroll    = config["scroll"]
        layer.z_index       = config["z"]
        
        var sprite = Sprite2D.new()
        sprite.texture  = load(config["texture"])
        sprite.centered = false
        layer.add_child(sprite)
        add_child(layer)
```

---

## 12. 从旧节点迁移

### 12.1 迁移步骤

1. **删除** `ParallaxBackground` 节点
2. **将** `ParallaxLayer` 替换为 `Parallax2D`
3. **映射属性**（见下表）
4. 如有代码引用 `ParallaxBackground.scroll_offset`，改为直接操作各 `Parallax2D.scroll_offset`

### 12.2 属性映射代码示例

**旧代码：**
```gdscript
# 旧方式
$ParallaxBackground.scroll_offset.x += speed * delta
$ParallaxBackground/SkyLayer.motion_scale = Vector2(0.1, 0.1)
$ParallaxBackground/SkyLayer.motion_mirroring = Vector2(1024, 0)
```

**新代码：**
```gdscript
# 新方式
$SkyLayer.scroll_offset.x += speed * delta   # 或使用 autoscroll
$SkyLayer.scroll_scale = Vector2(0.1, 0.1)
$SkyLayer.repeat_size  = Vector2(1024, 0)
```

---

## 13. 常见问题与陷阱

### Q1：图层出现接缝/跳变

**原因：** `repeat_size` 与图像实际尺寸不匹配。

**解决：**
```gdscript
# 获取图像实际宽度并设置
var texture_width = $Layer/Sprite2D.texture.get_width()
$Layer.repeat_size = Vector2(texture_width, 0)
```

---

### Q2：视差效果方向反了

**原因：** `scroll_scale` 理解有误。

**说明：** 当相机向右移动时，`scroll_scale < 1` 的图层移动距离小于相机，视觉上产生"落后"效果，这是正确的远景表现。

```gdscript
# 不要设为负值来"修正"方向，而是检查相机移动方向
# 如果效果真的反了，检查是否误用了负数
$Layer.scroll_scale = Vector2(0.3, 0.0)  # 正确
```

---

### Q3：autoscroll 不生效

**原因：** `repeat_size` 为 `(0, 0)` 时，自动滚动超出单张图后会停止。

**解决：** 必须同时设置 `repeat_size`：
```gdscript
$Layer.autoscroll  = Vector2(-30, 0)
$Layer.repeat_size = Vector2(1024, 0)  # 不能忘记！
```

---

### Q4：图层 Z 轴遮挡顺序错误

**原因：** 未设置 `z_index`。

**解决：** 近景图层设置更大的 `z_index`：
```gdscript
$SkyLayer.z_index      = -20
$MountainLayer.z_index = -10
$ForestLayer.z_index   = -5
$GroundLayer.z_index   =  0
```

---

### Q5：在没有 Camera2D 的情况下使用

`Parallax2D` 默认跟随视口的 `camera_2d`。没有 Camera2D 时：
- `scroll_scale` 的视差效果不会生效
- `autoscroll` 依然正常工作
- 可手动通过 `scroll_offset` 驱动

---

### Q6：Sprite2D 的 centered 属性影响循环

建议将子 `Sprite2D` 的 `centered` 设为 `false`，并将 Sprite 放置于 `(0, 0)`，这样 `repeat_size` 的计算更直观准确。

---

### Q7：Godot 4.2 及以下没有 Parallax2D

`Parallax2D` 仅在 **Godot 4.3+** 可用。如需兼容旧版本，使用 `ParallaxBackground + ParallaxLayer`。

---

## 14. 完整项目示例

### 14.1 横版游戏完整背景系统

```gdscript
# parallax_background.gd
# 挂载在包含所有 Parallax2D 层的 Node2D 上
extends Node2D

@export var enable_mouse_parallax: bool = false
@export var mouse_strength: float = 20.0

@onready var sky_layer      : Parallax2D = $SkyLayer
@onready var cloud_layer    : Parallax2D = $CloudLayer
@onready var mountain_layer : Parallax2D = $MountainLayer
@onready var forest_layer   : Parallax2D = $ForestLayer

var base_offsets: Dictionary = {}
var smooth_speed: float = 4.0
var target_mouse_offset: Vector2 = Vector2.ZERO
var current_mouse_offset: Vector2 = Vector2.ZERO

func _ready() -> void:
    _setup_layers()
    _save_base_offsets()

func _setup_layers() -> void:
    # 天空：最远，几乎不动，自动缓慢漂移
    sky_layer.scroll_scale = Vector2(0.02, 0.0)
    sky_layer.autoscroll   = Vector2(-5, 0)
    sky_layer.repeat_size  = Vector2(2048, 0)
    
    # 云彩：较远，自动飘动
    cloud_layer.scroll_scale = Vector2(0.1, 0.0)
    cloud_layer.autoscroll   = Vector2(-20, 0)
    cloud_layer.repeat_size  = Vector2(1024, 0)
    
    # 远山：慢速跟随相机
    mountain_layer.scroll_scale = Vector2(0.2, 0.05)
    mountain_layer.repeat_size  = Vector2(1024, 0)
    
    # 近处森林：较快跟随相机
    forest_layer.scroll_scale = Vector2(0.5, 0.0)
    forest_layer.repeat_size  = Vector2(512, 0)

func _save_base_offsets() -> void:
    for layer in [sky_layer, cloud_layer, mountain_layer, forest_layer]:
        base_offsets[layer] = layer.scroll_offset

func _process(delta: float) -> void:
    if enable_mouse_parallax:
        _update_mouse_parallax(delta)

func _update_mouse_parallax(delta: float) -> void:
    var viewport_size = get_viewport_rect().size
    var mouse_pos = get_viewport().get_mouse_position()
    var normalized = (mouse_pos / viewport_size) * 2.0 - Vector2.ONE
    
    target_mouse_offset = normalized * mouse_strength
    current_mouse_offset = current_mouse_offset.lerp(
        target_mouse_offset, smooth_speed * delta
    )
    
    var strengths = {
        sky_layer:      0.2,
        cloud_layer:    0.5,
        mountain_layer: 1.0,
        forest_layer:   2.0,
    }
    
    for layer in strengths:
        layer.scroll_offset = (
            base_offsets[layer] +
            current_mouse_offset * strengths[layer]
        )

# 外部调用：屏幕震动
func screen_shake(intensity: float = 15.0, duration: float = 0.4) -> void:
    var tween = create_tween()
    var steps = 8
    var step_time = duration / steps
    
    for i in steps:
        var offset = Vector2(
            randf_range(-intensity, intensity),
            randf_range(-intensity, intensity)
        )
        var decay = float(steps - i) / steps
        tween.tween_callback(func():
            for layer in [sky_layer, cloud_layer, mountain_layer, forest_layer]:
                layer.scroll_offset = base_offsets[layer] + offset * decay
        )
        tween.tween_interval(step_time)
    
    tween.tween_callback(func():
        for layer in [sky_layer, cloud_layer, mountain_layer, forest_layer]:
            layer.scroll_offset = base_offsets[layer]
    )
```

### 14.2 场景文件结构

```
game.tscn
├── ParallaxBackground (Node2D) [parallax_background.gd]
│   ├── Parallax2D [SkyLayer]
│   │   └── Sprite2D  centered=false  texture=sky.png
│   ├── Parallax2D [CloudLayer]
│   │   └── Sprite2D  centered=false  texture=clouds.png
│   ├── Parallax2D [MountainLayer]
│   │   └── Sprite2D  centered=false  texture=mountain.png
│   └── Parallax2D [ForestLayer]
│       └── Sprite2D  centered=false  texture=forest.png
├── TileMapLayer [地图]
├── Player (CharacterBody2D)
└── Camera2D  (process_callback=Idle, enabled=true)
```

---

## 15. 性能优化建议

### 15.1 控制图层数量

视差图层不是越多越好，建议 **3 ~ 5 层**为宜。过多图层会增加绘制调用（draw call）。

### 15.2 使用合适的图像格式

- 背景图使用 **无透明通道的 JPG** 可减少内存和绘制开销
- 有透明度的近景使用 **PNG**，但避免过大尺寸

### 15.3 repeat_size 与图像匹配

`repeat_size` 精确匹配图像尺寸可避免 GPU 进行不必要的采样计算。

### 15.4 关闭不可见图层的处理

```gdscript
# 当场景不可见时暂停自动滚动
func _on_visibility_changed() -> void:
    var layers = [$SkyLayer, $CloudLayer]
    for layer in layers:
        layer.set_process(visible)
```

### 15.5 CanvasLayer 分离 UI

如果有 UI 元素，使用 `CanvasLayer` 与视差背景隔离，避免 UI 受视差偏移影响：

```
Root
├── CanvasLayer (layer=-1) ← 视差背景
│   └── [所有 Parallax2D]
├── [游戏主体内容]
└── CanvasLayer (layer=1)  ← UI/HUD
    └── [血条、分数等]
```

---

## 附录：属性速查表

| 属性 | 类型 | 默认值 | 一句话描述 |
|------|------|--------|-----------|
| `scroll_scale` | `Vector2` | `(1,1)` | 视差速度比例，越小越远 |
| `scroll_offset` | `Vector2` | `(0,0)` | 手动偏移量 |
| `repeat_size` | `Vector2` | `(0,0)` | 无缝循环尺寸，0 表示不循环 |
| `autoscroll` | `Vector2` | `(0,0)` | 自动滚动速度（px/s） |
| `screen_offset` | `Vector2` | `(0,0)` | 相对屏幕的偏移 |
| `limit_begin` | `Vector2` | `(-1e7,-1e7)` | 滚动范围下限 |
| `limit_end` | `Vector2` | `(1e7,1e7)` | 滚动范围上限 |
| `follow_viewport` | `bool` | `true` | 是否跟随相机 |

---
