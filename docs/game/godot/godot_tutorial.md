# Godot

## 1. 引擎介绍与安装

### 1.1 什么是 Godot？

Godot 是一款**完全开源、免费**的跨平台游戏引擎，支持 2D 和 3D 游戏开发。

主要特点：
- 使用节点（Node）与场景（Scene）组合系统，结构清晰
- 内置 GDScript 脚本语言（语法类似 Python），上手极快
- 同时支持 C#、C++ 扩展
- 导出目标覆盖 Windows、macOS、Linux、Android、iOS、Web
- 编辑器本身非常轻量（安装包约 100MB）

### 1.2 下载与安装

1. 访问官网：https://godotengine.org
2. 下载 **Godot 4.x Standard** 版本（不含 .NET，纯 GDScript）
3. 解压即可运行，**无需安装**
4. 推荐将可执行文件添加到系统 PATH

### 1.3 版本说明

| 版本 | 说明 |
|------|------|
| Godot 4.x Standard | 推荐，支持 GDScript / C++ |
| Godot 4.x .NET | 支持 C# |
| Godot 3.x | 旧版，项目迁移需注意 API 差异 |

---

## 2. 编辑器界面详解

```
┌─────────────────────────────────────────────────────┐
│  菜单栏 (Scene / Project / Debug / Editor / Help)    │
├──────────┬──────────────────────────┬───────────────┤
│          │                          │               │
│ 场景树   │      视口（Viewport）     │  检查器       │
│ (Scene)  │      3D / 2D / Script    │ (Inspector)   │
│          │                          │               │
├──────────┴──────────────────────────┴───────────────┤
│   文件系统（FileSystem）  |  输出（Output）          │
└─────────────────────────────────────────────────────┘
```

### 2.1 主要面板说明

| 面板 | 快捷键 | 功能 |
|------|--------|------|
| 场景树 | - | 显示当前场景的节点层级结构 |
| 视口 | F1/F2/F3 | 切换 2D/3D/Script 视图 |
| 检查器 | - | 编辑选中节点的属性 |
| 文件系统 | - | 管理项目资源文件 |
| 输出面板 | - | 查看日志、错误、调试信息 |

### 2.2 常用快捷键

| 操作 | 快捷键 |
|------|--------|
| 运行项目 | F5 |
| 运行当前场景 | F6 |
| 停止运行 | F8 |
| 保存场景 | Ctrl+S |
| 切换 2D 视图 | Ctrl+F1 |
| 切换 3D 视图 | Ctrl+F2 |
| 切换脚本编辑器 | Ctrl+F3 |
| 搜索节点类型 | Ctrl+A |
| 搜索并打开文件 | Ctrl+Shift+O |

---

## 3. 场景与节点系统

### 3.1 核心概念

Godot 的一切都由**节点（Node）**构成，多个节点组合成**场景（Scene）**。

```
场景 (Scene .tscn)
└── 根节点 (Root Node)
    ├── 子节点 A
    │   └── 孙节点 A1
    └── 子节点 B
```

- 每个 .tscn 文件就是一个场景
- 场景可以嵌套（实例化）到其他场景中
- 每个节点只有一个父节点

### 3.2 常用节点类型

#### 通用节点

| 节点 | 用途 |
|------|------|
| `Node` | 最基础节点，无视觉表现 |
| `Node2D` | 2D 场景的基础节点，有位置/旋转/缩放 |
| `Node3D` | 3D 场景的基础节点 |
| `Control` | UI 控件基础节点 |

#### 2D 节点

| 节点 | 用途 |
|------|------|
| `Sprite2D` | 显示 2D 图片 |
| `AnimatedSprite2D` | 帧动画精灵 |
| `CharacterBody2D` | 角色物理体（可移动） |
| `StaticBody2D` | 静态物理体（地面墙壁） |
| `RigidBody2D` | 刚体（受重力） |
| `Area2D` | 检测重叠区域 |
| `CollisionShape2D` | 碰撞形状 |
| `Camera2D` | 2D 摄像机 |
| `TileMap` | 瓦片地图 |

#### 3D 节点

| 节点 | 用途 |
|------|------|
| `MeshInstance3D` | 显示 3D 网格 |
| `CharacterBody3D` | 角色物理体 |
| `StaticBody3D` | 静态物理体 |
| `RigidBody3D` | 3D 刚体 |
| `Camera3D` | 3D 摄像机 |
| `DirectionalLight3D` | 方向光（模拟太阳） |
| `OmniLight3D` | 点光源 |
| `SpotLight3D` | 聚光灯 |

### 3.3 创建第一个场景

1. 点击"场景"菜单 → 新建场景
2. 选择根节点类型（如 `Node2D`）
3. 右键根节点 → 添加子节点
4. 搜索并添加 `Sprite2D`
5. 在检查器中给 Sprite2D 指定贴图（Texture）
6. Ctrl+S 保存为 `main.tscn`

### 3.4 场景实例化

```gdscript
# 方法一：代码中实例化
var scene = preload("res://enemy.tscn")
var enemy = scene.instantiate()
add_child(enemy)
enemy.position = Vector2(100, 200)

# 方法二：编辑器中拖入
# 直接把 .tscn 文件拖到场景树即可
```

---

## 4. GDScript 编程基础

### 4.1 脚本挂载方式

右键任意节点 → 附加脚本，会生成如下模板：

```gdscript
extends Node2D  # 继承的节点类型

# 游戏开始时调用一次
func _ready():
    pass

# 每帧调用（delta 是上一帧到当前帧的时间，单位秒）
func _process(delta):
    pass
```

### 4.2 变量与类型

```gdscript
# 基本类型（GDScript 是动态类型，但推荐加类型注解）
var health: int = 100
var speed: float = 200.0
var name: String = "Player"
var is_alive: bool = true

# 常量
const MAX_HEALTH: int = 100
const GRAVITY: float = 980.0

# 向量（最常用的类型之一）
var pos2d: Vector2 = Vector2(10, 20)   # 2D 坐标
var pos3d: Vector3 = Vector3(1, 2, 3)  # 3D 坐标

# 数组
var items: Array = ["sword", "shield"]
var typed_array: Array[int] = [1, 2, 3]

# 字典
var stats: Dictionary = {
    "attack": 10,
    "defense": 5
}

# 枚举
enum State { IDLE, WALK, JUMP, DEAD }
var current_state: State = State.IDLE
```

### 4.3 控制流

```gdscript
# if / elif / else
if health > 50:
    print("健康")
elif health > 0:
    print("受伤")
else:
    print("死亡")

# match（类似 switch）
match current_state:
    State.IDLE:
        print("站立")
    State.WALK:
        print("行走")
    _:  # 默认分支
        print("其他状态")

# for 循环
for i in range(5):        # 0, 1, 2, 3, 4
    print(i)

for item in items:
    print(item)

# while 循环
var count = 0
while count < 10:
    count += 1

# 跳出与继续
for i in range(10):
    if i == 5:
        break       # 跳出循环
    if i % 2 == 0:
        continue    # 跳过偶数
    print(i)
```

### 4.4 函数

```gdscript
# 基本函数
func greet(player_name: String) -> String:
    return "Hello, " + player_name

# 默认参数
func move(direction: Vector2, speed: float = 100.0) -> void:
    position += direction * speed

# 可变参数
func sum(*numbers) -> float:
    var total = 0.0
    for n in numbers:
        total += n
    return total

# Lambda（匿名函数）
var double = func(x): return x * 2
print(double.call(5))  # 输出 10
```

### 4.5 类与继承

```gdscript
# 定义类（每个脚本文件即一个类）
class_name Enemy extends CharacterBody2D

# 成员变量
var health: int = 50
var attack_power: int = 10

# 构造函数
func _init(hp: int = 50):
    health = hp

# 方法
func take_damage(amount: int) -> void:
    health -= amount
    if health <= 0:
        die()

func die() -> void:
    queue_free()  # 从场景中删除该节点

# 内部类
class Loot:
    var gold: int
    var items: Array

# 继承
class BossEnemy extends Enemy:
    var phase: int = 1
    
    func _init():
        super(200)  # 调用父类构造函数
    
    func take_damage(amount: int) -> void:
        super(amount / 2)  # 调用父类方法（Boss 减少伤害）
```

### 4.6 常用内置函数

```gdscript
# 输出调试信息
print("Hello")
print_debug("带行号的调试信息")
push_warning("警告信息")
push_error("错误信息")

# 类型转换
var f: float = float(10)
var i: int = int(3.7)        # 截断为 3
var s: String = str(42)

# 数学函数
var a = abs(-5)              # 5
var b = clamp(15, 0, 10)    # 10（限制在 0~10 范围）
var c = lerp(0.0, 100.0, 0.5)  # 50.0（线性插值）
var d = sqrt(16.0)           # 4.0
var e = pow(2.0, 10.0)       # 1024.0
var f2 = randf()              # 0.0~1.0 随机浮点
var g = randi() % 10         # 0~9 随机整数
var h = snappedf(3.7, 0.5)   # 4.0（吸附到 0.5 的倍数）

# 向量操作
var v = Vector2(3, 4)
print(v.length())            # 5.0（长度）
print(v.normalized())        # (0.6, 0.8)（归一化）
print(v.dot(Vector2(1, 0)))  # 3.0（点积）
```

---

## 5. 信号系统（Signals）

信号是 Godot 的观察者模式实现，用于节点间通信，避免硬耦合。

### 5.1 使用内置信号

```gdscript
# 在编辑器中连接：选中节点 → 节点面板 → 信号列表 → 双击连接

# 或在代码中连接
func _ready():
    # 连接按钮的 pressed 信号到本脚本的函数
    $Button.pressed.connect(_on_button_pressed)
    
    # 连接 Area2D 的 body_entered 信号
    $Area2D.body_entered.connect(_on_area_body_entered)

func _on_button_pressed():
    print("按钮被点击了")

func _on_area_body_entered(body: Node2D):
    print(body.name, "进入了区域")
```

### 5.2 自定义信号

```gdscript
class_name Player extends CharacterBody2D

# 声明信号（可以带参数）
signal health_changed(new_health: int, old_health: int)
signal died

var _health: int = 100

func take_damage(amount: int) -> void:
    var old_hp = _health
    _health = max(0, _health - amount)
    
    # 发射信号
    health_changed.emit(_health, old_hp)
    
    if _health == 0:
        died.emit()
```

```gdscript
# 在 UI 脚本中监听
func _ready():
    var player = $Player
    player.health_changed.connect(_on_player_health_changed)
    player.died.connect(_on_player_died)

func _on_player_health_changed(new_hp: int, old_hp: int):
    $HealthBar.value = new_hp

func _on_player_died():
    $GameOverScreen.show()
```

### 5.3 一次性信号连接

```gdscript
# 信号触发一次后自动断开
$Timer.timeout.connect(_on_timer_done, CONNECT_ONE_SHOT)
```

---

## 6. 2D 游戏开发

### 6.1 角色移动

```gdscript
extends CharacterBody2D

const SPEED = 200.0
const JUMP_VELOCITY = -400.0
const GRAVITY = 980.0

func _physics_process(delta: float) -> void:
    # 重力
    if not is_on_floor():
        velocity.y += GRAVITY * delta

    # 跳跃
    if Input.is_action_just_pressed("ui_accept") and is_on_floor():
        velocity.y = JUMP_VELOCITY

    # 水平移动
    var direction = Input.get_axis("ui_left", "ui_right")
    if direction != 0:
        velocity.x = direction * SPEED
    else:
        velocity.x = move_toward(velocity.x, 0, SPEED)

    move_and_slide()
```

### 6.2 输入映射

在 项目设置 → 输入映射 中配置，或代码中查询：

```gdscript
# 检测按键状态
Input.is_action_pressed("jump")       # 持续按住
Input.is_action_just_pressed("jump")  # 刚刚按下（单帧）
Input.is_action_just_released("jump") # 刚刚松开（单帧）

# 获取轴值（-1 到 1）
var h = Input.get_axis("move_left", "move_right")
var v = Input.get_axis("move_up", "move_down")

# 获取向量（自动归一化）
var dir = Input.get_vector("move_left", "move_right", "move_up", "move_down")
```

### 6.3 精灵翻转

```gdscript
# 根据移动方向翻转角色
func _physics_process(delta):
    var direction = Input.get_axis("ui_left", "ui_right")
    if direction > 0:
        $Sprite2D.flip_h = false
    elif direction < 0:
        $Sprite2D.flip_h = true
```

### 6.4 摄像机跟随

```gdscript
# 将 Camera2D 作为 Player 的子节点即可自动跟随
# 常用属性：
# Position Smoothing Enabled = true（平滑跟随）
# Limit（设置摄像机边界，防止看到关卡外）
```

### 6.5 碰撞检测（Area2D）

```gdscript
extends Area2D

func _ready():
    body_entered.connect(_on_body_entered)
    body_exited.connect(_on_body_exited)

func _on_body_entered(body: Node2D):
    if body.is_in_group("player"):
        body.collect_item()
        queue_free()

func _on_body_exited(body: Node2D):
    pass
```

### 6.6 TileMap 地图系统

```
1. 在 FileSystem 中创建 TileSet 资源
2. 添加 TileMap 节点到场景
3. 在检查器中指定 TileSet
4. 选择画刷，在视口中绘制地图
5. 给 TileSet 中的 Tile 添加碰撞形状以实现物理
```

---

## 7. 3D 游戏开发

### 7.1 3D 场景基础

```gdscript
extends CharacterBody3D

const SPEED = 5.0
const JUMP_VELOCITY = 4.5
const GRAVITY = 9.8

@onready var camera = $Camera3D

func _physics_process(delta: float) -> void:
    if not is_on_floor():
        velocity.y -= GRAVITY * delta

    if Input.is_action_just_pressed("ui_accept") and is_on_floor():
        velocity.y = JUMP_VELOCITY

    var input_dir = Input.get_vector("ui_left", "ui_right", "ui_up", "ui_down")
    
    # 相对摄像机方向移动
    var direction = (camera.transform.basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()
    
    if direction:
        velocity.x = direction.x * SPEED
        velocity.z = direction.z * SPEED
    else:
        velocity.x = move_toward(velocity.x, 0, SPEED)
        velocity.z = move_toward(velocity.z, 0, SPEED)

    move_and_slide()
```

### 7.2 鼠标视角控制

```gdscript
extends Node3D

@export var mouse_sensitivity: float = 0.003

@onready var camera_pivot = $CameraPivot
@onready var camera = $CameraPivot/Camera3D

func _ready():
    Input.set_mouse_mode(Input.MOUSE_MODE_CAPTURED)

func _unhandled_input(event: InputEvent) -> void:
    if event is InputEventMouseMotion:
        rotate_y(-event.relative.x * mouse_sensitivity)
        camera_pivot.rotate_x(-event.relative.y * mouse_sensitivity)
        # 限制俯仰角
        camera_pivot.rotation.x = clamp(camera_pivot.rotation.x, -PI/2, PI/2)
    
    if event.is_action_pressed("ui_cancel"):
        Input.set_mouse_mode(Input.MOUSE_MODE_VISIBLE)
```

### 7.3 3D 模型导入

支持格式：`.glb`（推荐）、`.gltf`、`.fbx`、`.obj`

```
1. 将模型文件放入 res:// 目录
2. Godot 自动导入，生成 .import 文件
3. 将模型拖入场景，或在代码中加载：
   var model = load("res://models/character.glb").instantiate()
```

---

## 8. 物理系统

### 8.1 物理体类型对比

| 节点 | 移动方式 | 适用场景 |
|------|----------|----------|
| `StaticBody2D/3D` | 不移动 | 地面、墙壁、平台 |
| `CharacterBody2D/3D` | `move_and_slide()` | 玩家、NPC |
| `RigidBody2D/3D` | 物理引擎控制 | 箱子、抛出物 |
| `Area2D/3D` | 无物理，仅检测 | 触发区、传感器 |

### 8.2 RigidBody 控制

```gdscript
extends RigidBody2D

func _ready():
    # 施加力
    apply_force(Vector2(0, -500))     # 持续力
    apply_impulse(Vector2(100, -200)) # 瞬间冲量

func _integrate_forces(state: PhysicsDirectBodyState2D):
    # 精确控制（在物理步中调用）
    state.linear_velocity = Vector2(100, state.linear_velocity.y)
```

### 8.3 碰撞层设置

```
物理层（Layer）：该对象存在于哪层
碰撞遮罩（Mask）：与哪些层发生碰撞

示例：
  玩家  Layer=1, Mask=2（地形层）+4（敌人层）
  地形  Layer=2, Mask=0
  敌人  Layer=4, Mask=1（玩家层）+2（地形层）
```

```gdscript
# 代码中设置
collision_layer = 1   # 第 1 层
collision_mask = 2    # 与第 2 层碰撞

# 使用位运算设置多层
collision_mask = (1 << 1) | (1 << 2)  # 第 2 层和第 3 层
```

### 8.4 射线检测

```gdscript
func shoot_ray(from: Vector2, direction: Vector2, length: float):
    var space = get_world_2d().direct_space_state
    var query = PhysicsRayQueryParameters2D.create(
        from,
        from + direction * length
    )
    query.exclude = [self]  # 排除自身
    
    var result = space.intersect_ray(query)
    if result:
        print("击中: ", result.collider.name)
        print("位置: ", result.position)
        print("法线: ", result.normal)
```

---

## 9. 动画系统

### 9.1 AnimationPlayer

AnimationPlayer 可以为任何节点属性设置关键帧动画。

```gdscript
extends Node2D

@onready var anim = $AnimationPlayer

func _ready():
    # 播放动画
    anim.play("walk")
    
    # 播放并从头开始
    anim.play("attack")
    
    # 停止
    anim.stop()
    
    # 设置播放速度
    anim.speed_scale = 1.5
    
    # 监听动画结束
    anim.animation_finished.connect(_on_animation_finished)

func _on_animation_finished(anim_name: StringName):
    if anim_name == "attack":
        anim.play("idle")
```

### 9.2 AnimatedSprite2D

专门用于帧序列动画（精灵表）：

```gdscript
extends CharacterBody2D

@onready var sprite = $AnimatedSprite2D

func _physics_process(delta):
    var direction = Input.get_axis("ui_left", "ui_right")
    
    if is_on_floor():
        if direction == 0:
            sprite.play("idle")
        else:
            sprite.play("walk")
    else:
        sprite.play("jump")
```

### 9.3 AnimationTree（状态机）

AnimationTree 提供动画状态机，适合复杂动画逻辑：

```gdscript
@onready var anim_tree = $AnimationTree
@onready var state_machine = anim_tree.get("parameters/playback")

func _ready():
    anim_tree.active = true

func update_animation(is_running: bool, is_jumping: bool):
    if is_jumping:
        state_machine.travel("jump")
    elif is_running:
        state_machine.travel("run")
    else:
        state_machine.travel("idle")

# 设置混合参数（用于 BlendSpace）
anim_tree.set("parameters/blend_position", velocity.length() / MAX_SPEED)
```

### 9.4 Tween（补间动画）

用于平滑过渡属性值，无需手动设关键帧：

```gdscript
func fade_out():
    var tween = create_tween()
    tween.tween_property(self, "modulate:a", 0.0, 1.0)  # 1秒淡出

func bounce_in():
    var tween = create_tween()
    tween.set_ease(Tween.EASE_OUT)
    tween.set_trans(Tween.TRANS_BOUNCE)
    tween.tween_property($Sprite, "scale", Vector2(1, 1), 0.5)\
         .from(Vector2(0, 0))

func sequence_animation():
    var tween = create_tween()
    # 链式动画
    tween.tween_property($Icon, "position", Vector2(200, 0), 1.0)
    tween.tween_interval(0.5)  # 等待 0.5 秒
    tween.tween_property($Icon, "position", Vector2(0, 0), 1.0)
    tween.tween_callback(func(): print("动画完成"))
```

---

## 10. UI 与用户界面

### 10.1 Control 节点基础

所有 UI 元素都继承自 `Control` 节点：

| 节点 | 用途 |
|------|------|
| `Label` | 显示文字 |
| `Button` | 按钮 |
| `TextEdit` | 多行文本输入 |
| `LineEdit` | 单行文本输入 |
| `ProgressBar` | 进度条 |
| `Slider` | 滑动条 |
| `CheckBox` | 复选框 |
| `OptionButton` | 下拉选项 |
| `Panel` | 容器面板 |
| `VBoxContainer` | 垂直排列容器 |
| `HBoxContainer` | 水平排列容器 |
| `GridContainer` | 网格容器 |
| `ScrollContainer` | 可滚动容器 |
| `TabContainer` | 标签页容器 |

### 10.2 锚点与边距

```
锚点（Anchor）：相对父容器的基准点（0~1）
  锚点 (0,0) = 左上角
  锚点 (1,1) = 右下角
  锚点 (0.5,0.5) = 中心

边距（Offset）：相对锚点的像素偏移
```

常用布局预设（检查器 Layout 按钮）：
- Full Rect：填满父容器
- Top Wide：顶部横条（如顶部菜单栏）
- Bottom Wide：底部横条（如状态栏）

### 10.3 常用 UI 代码

```gdscript
# Label
$Label.text = "Score: %d" % score
$Label.add_theme_color_override("font_color", Color.RED)

# Button
$Button.text = "开始游戏"
$Button.pressed.connect(func(): get_tree().change_scene_to_file("res://game.tscn"))

# ProgressBar（血条）
$HealthBar.max_value = 100
$HealthBar.value = current_health

# 动态创建 UI 元素
var label = Label.new()
label.text = "动态文字"
$VBoxContainer.add_child(label)
```

### 10.4 主题与样式

```gdscript
# 代码中修改样式
var style = StyleBoxFlat.new()
style.bg_color = Color(0.2, 0.2, 0.2, 0.8)
style.corner_radius_top_left = 8
style.corner_radius_top_right = 8
$Panel.add_theme_stylebox_override("panel", style)

# 字体大小
$Label.add_theme_font_size_override("font_size", 24)
```

---

## 11. 音频系统

### 11.1 节点类型

| 节点 | 用途 |
|------|------|
| `AudioStreamPlayer` | 播放全局音效（BGM、UI 音效） |
| `AudioStreamPlayer2D` | 2D 空间音效（有距离衰减） |
| `AudioStreamPlayer3D` | 3D 空间音效 |

### 11.2 基本用法

```gdscript
# 播放 BGM
@onready var bgm = $AudioStreamPlayer

func _ready():
    bgm.stream = load("res://audio/bgm.ogg")
    bgm.volume_db = -10.0  # 音量（分贝）
    bgm.play()

# 播放一次性音效
func play_sfx(sound_path: String):
    var player = AudioStreamPlayer.new()
    player.stream = load(sound_path)
    player.autoplay = true
    player.finished.connect(player.queue_free)  # 播放完自动删除
    add_child(player)

# 2D 音效（距离衰减）
func play_explosion(pos: Vector2):
    var player = AudioStreamPlayer2D.new()
    player.stream = preload("res://audio/explosion.wav")
    player.position = pos
    player.max_distance = 500.0
    add_child(player)
    player.play()
```

### 11.3 音频总线

在 **音频** 面板中配置音频总线（Master、Music、SFX）：

```gdscript
# 调整总线音量
AudioServer.set_bus_volume_db(AudioServer.get_bus_index("Music"), -20.0)

# 静音总线
AudioServer.set_bus_mute(AudioServer.get_bus_index("SFX"), true)
```

---

## 12. 资源管理

### 12.1 资源加载方式

```gdscript
# preload：编译时加载（推荐用于确定存在的小资源）
var texture = preload("res://sprites/player.png")

# load：运行时加载（适合动态加载）
var scene = load("res://scenes/enemy.tscn")

# 异步加载（大型资源，避免卡顿）
func _ready():
    ResourceLoader.load_threaded_request("res://levels/level2.tscn")

func _process(_delta):
    var status = ResourceLoader.load_threaded_get_status("res://levels/level2.tscn")
    if status == ResourceLoader.THREAD_LOAD_LOADED:
        var scene = ResourceLoader.load_threaded_get("res://levels/level2.tscn")
        get_tree().change_scene_to_packed(scene)
```

### 12.2 自定义资源

```gdscript
# 定义可保存的数据资源
class_name CharacterData extends Resource

@export var character_name: String = ""
@export var max_health: int = 100
@export var attack_power: int = 10
@export var sprite: Texture2D

# 在编辑器中：右键 FileSystem → 新建资源 → CharacterData
# 使用代码保存：
func save_data():
    var data = CharacterData.new()
    data.character_name = "Hero"
    data.max_health = 150
    ResourceSaver.save(data, "res://data/hero.tres")

# 加载自定义资源：
var hero_data: CharacterData = load("res://data/hero.tres")
```

### 12.3 存档系统

```gdscript
const SAVE_PATH = "user://save_game.cfg"

func save_game():
    var config = ConfigFile.new()
    config.set_value("player", "health", player.health)
    config.set_value("player", "position_x", player.position.x)
    config.set_value("player", "position_y", player.position.y)
    config.set_value("game", "score", score)
    config.save(SAVE_PATH)

func load_game():
    var config = ConfigFile.new()
    var err = config.load(SAVE_PATH)
    if err != OK:
        print("没有存档文件")
        return
    
    player.health = config.get_value("player", "health", 100)
    player.position.x = config.get_value("player", "position_x", 0.0)
    player.position.y = config.get_value("player", "position_y", 0.0)
    score = config.get_value("game", "score", 0)
```

### 12.4 场景切换

```gdscript
# 切换到新场景
get_tree().change_scene_to_file("res://scenes/game_over.tscn")

# 带过渡切换（需要自定义过渡效果）
func change_scene_with_fade(scene_path: String):
    var tween = create_tween()
    tween.tween_property($FadeOverlay, "modulate:a", 1.0, 0.5)
    tween.tween_callback(func():
        get_tree().change_scene_to_file(scene_path)
    )

# 重载当前场景
get_tree().reload_current_scene()
```

---

## 13. 项目导出与发布

### 13.1 安装导出模板

1. 打开 Godot → 编辑器 → 管理导出模板
2. 下载对应版本的导出模板
3. 安装完成后即可导出到对应平台

### 13.2 导出到各平台

**Windows/macOS/Linux：**
1. 项目 → 导出
2. 添加预设 → 选择平台
3. 配置图标、名称、权限
4. 点击"导出项目"

**Android：**
1. 安装 Android SDK 和 JDK
2. 在编辑器设置中配置 SDK 路径
3. 生成 keystore 签名文件
4. 导出 .apk 或 .aab

**Web（HTML5）：**
1. 添加 Web 导出预设
2. 导出后上传到 Web 服务器
3. 需要 HTTPS（或 itch.io 等支持平台）

### 13.3 导出注意事项

```
- 不要将 res:// 路径硬编码到导出平台不兼容的地方
- 用 user:// 存储运行时生成的文件（存档、日志）
- 检查资源大小，压缩贴图（启用 导入 → Compress）
- 关闭不需要的功能模块以减小包体
- 测试在目标平台真机上的运行效果
```

---

## 14. 实战项目：2D 平台跳跃游戏

### 14.1 项目结构

```
res://
├── scenes/
│   ├── main.tscn          # 主场景
│   ├── player.tscn        # 玩家
│   ├── enemy.tscn         # 敌人
│   └── ui/
│       └── hud.tscn       # HUD界面
├── scripts/
│   ├── player.gd
│   ├── enemy.gd
│   ├── game_manager.gd
│   └── hud.gd
├── sprites/
│   ├── player/
│   └── enemy/
└── audio/
    ├── bgm.ogg
    └── sfx/
```

### 14.2 玩家脚本

```gdscript
# scripts/player.gd
class_name Player extends CharacterBody2D

signal health_changed(new_health: int)
signal died

const SPEED = 200.0
const JUMP_VELOCITY = -450.0
const GRAVITY = 980.0
const MAX_HEALTH = 3

var health: int = MAX_HEALTH
var is_dead: bool = false
var invincible: bool = false

@onready var sprite = $AnimatedSprite2D
@onready var coyote_timer = $CoyoteTimer   # 土狼时间（离地后短暂可跳跃）
@onready var jump_buffer_timer = $JumpBufferTimer

func _physics_process(delta: float) -> void:
    if is_dead:
        return
    
    _apply_gravity(delta)
    _handle_jump()
    _handle_movement()
    _update_animation()
    move_and_slide()

func _apply_gravity(delta: float) -> void:
    if not is_on_floor():
        velocity.y += GRAVITY * delta
    else:
        coyote_timer.start()

func _handle_jump() -> void:
    if Input.is_action_just_pressed("jump"):
        jump_buffer_timer.start()
    
    var can_jump = is_on_floor() or not coyote_timer.is_stopped()
    var wants_jump = not jump_buffer_timer.is_stopped()
    
    if can_jump and wants_jump:
        velocity.y = JUMP_VELOCITY
        coyote_timer.stop()
        jump_buffer_timer.stop()
    
    # 松开跳跃键时降低跳跃高度（变速跳）
    if Input.is_action_just_released("jump") and velocity.y < 0:
        velocity.y *= 0.5

func _handle_movement() -> void:
    var direction = Input.get_axis("move_left", "move_right")
    velocity.x = direction * SPEED if direction != 0 else move_toward(velocity.x, 0, SPEED * 10)

func _update_animation() -> void:
    var direction = Input.get_axis("move_left", "move_right")
    if direction != 0:
        sprite.flip_h = direction < 0
    
    if not is_on_floor():
        sprite.play("jump" if velocity.y < 0 else "fall")
    elif direction != 0:
        sprite.play("run")
    else:
        sprite.play("idle")

func take_damage() -> void:
    if invincible or is_dead:
        return
    
    health -= 1
    health_changed.emit(health)
    
    if health <= 0:
        _die()
    else:
        _start_invincibility()

func _start_invincibility() -> void:
    invincible = true
    var tween = create_tween().set_loops(5)
    tween.tween_property(sprite, "modulate:a", 0.3, 0.1)
    tween.tween_property(sprite, "modulate:a", 1.0, 0.1)
    tween.finished.connect(func(): invincible = false)

func _die() -> void:
    is_dead = true
    sprite.play("die")
    died.emit()
```

### 14.3 敌人脚本

```gdscript
# scripts/enemy.gd
class_name Enemy extends CharacterBody2D

const SPEED = 80.0
const GRAVITY = 980.0

var direction: float = 1.0

@onready var sprite = $AnimatedSprite2D
@onready var wall_detector = $WallDetector
@onready var edge_detector = $EdgeDetector

func _physics_process(delta: float) -> void:
    velocity.y += GRAVITY * delta
    velocity.x = direction * SPEED
    
    if wall_detector.is_colliding() or not edge_detector.is_colliding():
        direction *= -1
        sprite.flip_h = direction < 0
    
    move_and_slide()
    sprite.play("walk")

func _on_hitbox_body_entered(body: Node2D) -> void:
    if body is Player:
        # 从上方踩到敌人
        if body.velocity.y > 0 and body.global_position.y < global_position.y:
            body.velocity.y = -300.0  # 踩到敌人后弹起
            queue_free()
        else:
            body.take_damage()
```

### 14.4 游戏管理器

```gdscript
# scripts/game_manager.gd
extends Node

signal score_changed(new_score: int)
signal game_over

var score: int = 0
var high_score: int = 0

func _ready():
    load_high_score()
    
    var player = get_tree().get_first_node_in_group("player")
    if player:
        player.died.connect(_on_player_died)

func add_score(points: int) -> void:
    score += points
    score_changed.emit(score)
    
    if score > high_score:
        high_score = score

func _on_player_died() -> void:
    save_high_score()
    await get_tree().create_timer(2.0).timeout
    game_over.emit()

func save_high_score() -> void:
    var config = ConfigFile.new()
    config.set_value("game", "high_score", high_score)
    config.save("user://save.cfg")

func load_high_score() -> void:
    var config = ConfigFile.new()
    if config.load("user://save.cfg") == OK:
        high_score = config.get_value("game", "high_score", 0)
```

### 14.5 HUD 脚本

```gdscript
# scripts/hud.gd
extends CanvasLayer

@onready var score_label = $ScoreLabel
@onready var health_container = $HealthContainer
@onready var game_over_screen = $GameOverScreen

func _ready():
    var game_manager = get_node("/root/GameManager")
    game_manager.score_changed.connect(_on_score_changed)
    game_manager.game_over.connect(_on_game_over)
    
    var player = get_tree().get_first_node_in_group("player")
    player.health_changed.connect(_on_health_changed)

func _on_score_changed(new_score: int) -> void:
    score_label.text = "Score: %d" % new_score

func _on_health_changed(new_health: int) -> void:
    for i in health_container.get_child_count():
        var heart = health_container.get_child(i)
        heart.visible = i < new_health

func _on_game_over() -> void:
    game_over_screen.show()
    var game_manager = get_node("/root/GameManager")
    $GameOverScreen/HighScoreLabel.text = "最高分: %d" % game_manager.high_score
    $GameOverScreen/RestartButton.pressed.connect(func():
        get_tree().reload_current_scene()
    )
```

---

## 附录：调试技巧

### 常用调试手段

```gdscript
# 在游戏中显示调试信息
func _process(_delta):
    DebugDraw2D.draw_string(Vector2(10, 10), "FPS: %d" % Engine.get_frames_per_second())

# 暂停游戏
get_tree().paused = true

# 慢动作调试
Engine.time_scale = 0.1

# 打印节点树
print_tree_pretty()

# 检查节点是否存在
if is_instance_valid(enemy):
    enemy.queue_free()
```

### 性能分析

- 打开 调试 → 打开监视器 查看 FPS、内存、绘制调用
- 使用 调试 → 打开分析器 找出性能瓶颈
- `_physics_process` 中避免复杂计算，使用时间累积降低频率