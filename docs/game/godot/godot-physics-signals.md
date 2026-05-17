# Godot Physics & Signals

---

## 1. 物理系统概览

Godot 4 内置两套物理引擎：

| 引擎              | 说明                                       |
| ----------------- | ------------------------------------------ |
| **Godot Physics** | 默认引擎，纯 GDScript/C++ 实现，跨平台稳定 |
| **Jolt Physics**  | 可选插件，性能更强，适合大型 3D 场景       |

物理世界由 **PhysicsServer3D** / **PhysicsServer2D** 在底层管理。每个物理帧（默认 60Hz）独立于渲染帧运行，通过 `_physics_process(delta)` 回调暴露给脚本。

```
渲染帧  _process(delta)          ← 用于视觉/UI/动画插值
物理帧  _physics_process(delta)  ← 用于移动/力/碰撞检测
```

> **原则**：所有涉及物理体移动、力、碰撞检测的逻辑必须写在 `_physics_process` 中，否则行为不稳定。

---

## 2. 物理节点类型

```
Node
└── CollisionObject2D / CollisionObject3D   ← 所有物理体的基类
    ├── Area2D / Area3D                     ← 检测区域，不参与物理模拟
    ├── PhysicsBody2D / PhysicsBody3D
    │   ├── StaticBody                      ← 不移动的碰撞体（地形、墙壁）
    │   ├── AnimatableBody                  ← 可被脚本/动画移动的静态体
    │   ├── RigidBody                       ← 全物理模拟（重力、力、碰撞响应）
    │   └── CharacterBody                   ← 玩家/NPC，手动控制移动
    └── VehicleBody3D                       ← 车辆专用（3D）
```

---

## 3. 碰撞形状与层级

### 3.1 CollisionShape

每个物理体必须至少挂载一个 **CollisionShape2D** 或 **CollisionShape3D** 子节点。

常用 Shape：

| Shape                   | 用途                                |
| ----------------------- | ----------------------------------- |
| `BoxShape3D`            | 箱体，性能最优                      |
| `SphereShape3D`         | 球体，旋转无影响                    |
| `CapsuleShape3D`        | 角色胶囊体                          |
| `ConcavePolygonShape3D` | 复杂静态地形（只能用于 StaticBody） |
| `ConvexPolygonShape3D`  | 复杂动态体（凸包）                  |

### 3.2 碰撞层与掩码

```
collision_layer  ← 该物体"属于"哪些层（我是谁）
collision_mask   ← 该物体"检测"哪些层（我看谁）
```

```gdscript
# 设置角色在第1层，检测第2、3层
$Player.collision_layer = 1        # 0b0001
$Player.collision_mask  = 0b0110   # 第2层 + 第3层
```

> 两个物体发生碰撞的条件：A 的 mask 包含 B 的 layer，**或** B 的 mask 包含 A 的 layer。

---

## 4. RigidBody3D / RigidBody2D

RigidBody 完全由物理引擎驱动，支持重力、冲量、力矩。

### 4.1 运动模式

```gdscript
# 通过 freeze_mode 和 freeze 属性控制
body.freeze = false                          # 默认：参与物理
body.freeze_mode = RigidBody3D.FREEZE_MODE_KINEMATIC  # 冻结后可手动控制位置
```

| `physics_material_override` 属性 | 说明             |
| -------------------------------- | ---------------- |
| `friction`                       | 摩擦力           |
| `bounce`                         | 弹性             |
| `rough` / `absorbent`            | 是否覆盖对方参数 |

### 4.2 施加力与冲量

```gdscript
func _physics_process(_delta):
    # 持续力（每帧累加）
    apply_force(Vector3(10, 0, 0))

    # 一次性冲量（用于跳跃、爆炸）
    apply_impulse(Vector3(0, 500, 0))

    # 扭矩（旋转）
    apply_torque(Vector3(0, 1, 0))
```

### 4.3 重要信号

```gdscript
# 碰撞发生时（需在 Inspector 中启用 contact_monitor 并设置 max_contacts_reported）
body.body_entered.connect(_on_body_entered)
body.body_exited.connect(_on_body_exited)

func _on_body_entered(body: Node):
    print("碰撞到: ", body.name)
```

### 4.4 \_integrate_forces — 精确控制

```gdscript
func _integrate_forces(state: PhysicsDirectBodyState3D):
    # state 包含当前速度、接触点、质量等
    var velocity = state.linear_velocity
    # 直接设置速度（绕过力的累加）
    state.linear_velocity = Vector3(5, velocity.y, 0)
```

---

## 5. CharacterBody3D / CharacterBody2D

CharacterBody 是做角色控制器的首选，不受物理引擎自动模拟，由脚本完全控制。

### 5.1 move_and_slide

```gdscript
extends CharacterBody3D

const SPEED = 5.0
const JUMP_VELOCITY = 4.5

func _physics_process(delta):
    # 添加重力
    if not is_on_floor():
        velocity += get_gravity() * delta

    # 跳跃
    if Input.is_action_just_pressed("ui_accept") and is_on_floor():
        velocity.y = JUMP_VELOCITY

    # 水平移动
    var dir = Input.get_vector("ui_left", "ui_right", "ui_up", "ui_down")
    velocity.x = dir.x * SPEED
    velocity.z = dir.y * SPEED

    move_and_slide()
```

### 5.2 move_and_collide（底层）

```gdscript
func _physics_process(delta):
    var collision = move_and_collide(velocity * delta)
    if collision:
        # 反弹
        velocity = velocity.bounce(collision.get_normal())
```

| 方法                 | 适用场景                    |
| -------------------- | --------------------------- |
| `move_and_slide()`   | 角色、NPC，自动处理坡面滑动 |
| `move_and_collide()` | 弹球、自定义碰撞响应        |

### 5.3 常用状态查询

```gdscript
is_on_floor()       # 是否站在地面
is_on_ceiling()     # 是否顶到天花板
is_on_wall()        # 是否贴墙
get_floor_normal()  # 地面法线（用于坡度计算）
get_last_slide_collision()  # 最后一次碰撞信息
```

---

## 6. Area3D / Area2D

Area 不参与物理碰撞响应，只做**重叠检测**，常用于触发器、伤害区域、拾取范围。

### 6.1 基本用法

```gdscript
extends Area3D

func _ready():
    body_entered.connect(_on_body_entered)
    body_exited.connect(_on_body_exited)
    area_entered.connect(_on_area_entered)

func _on_body_entered(body: Node3D):
    if body.is_in_group("Player"):
        print("玩家进入区域")

func _on_body_exited(body: Node3D):
    print("物体离开区域")
```

### 6.2 重力与阻尼覆盖

Area 可以覆盖区域内 RigidBody 的重力方向和大小：

```gdscript
# Inspector 中设置，或脚本：
$WaterArea.gravity = 2.0
$WaterArea.gravity_direction = Vector3(0, -1, 0)
$WaterArea.linear_damp = 3.0   # 模拟水的阻力
```

### 6.3 检测 Area 内的物体（主动查询）

```gdscript
func get_enemies_in_range() -> Array:
    return get_overlapping_bodies().filter(
        func(b): return b.is_in_group("Enemy")
    )
```

---

## 7. StaticBody 与 AnimatableBody

### StaticBody

完全静止，消耗最小，用于地形、墙壁、平台。

```gdscript
extends StaticBody3D
# 通常不需要脚本，直接在编辑器摆放
```

### AnimatableBody

可以被 `AnimationPlayer` 或脚本移动位置，会把运动速度传递给碰到的 RigidBody（如传送带）。

```gdscript
extends AnimatableBody3D

func _physics_process(delta):
    # 直接修改 position，引擎会计算速度并推动碰到的物体
    position.x += 2.0 * delta
```

---

## 8. 物理查询：Raycast 与 ShapeCast

### 8.1 RayCast 节点

```gdscript
@onready var ray = $RayCast3D

func _physics_process(_delta):
    if ray.is_colliding():
        var point  = ray.get_collision_point()
        var normal = ray.get_collision_normal()
        var obj    = ray.get_collider()
        print("打到: ", obj.name, " 法线: ", normal)
```

### 8.2 即时 Raycast（不需要节点）

```gdscript
func cast_ray(from: Vector3, to: Vector3):
    var space = get_world_3d().direct_space_state
    var query = PhysicsRayQueryParameters3D.create(from, to)
    query.exclude = [self]           # 排除自身
    query.collision_mask = 0b0010    # 只检测第2层

    var result = space.intersect_ray(query)
    if result:
        print("命中: ", result["collider"].name)
        print("位置: ", result["position"])
```

### 8.3 ShapeCast（形状扫描）

```gdscript
# 用 SphereShape3D 扫描前方
var query = PhysicsShapeQueryParameters3D.new()
query.shape = SphereShape3D.new()
query.shape.radius = 0.5
query.transform = global_transform
query.motion = Vector3(0, 0, -5)

var results = space.cast_motion(query)
# results[0] = 未碰撞的安全比例, results[1] = 碰撞时的比例
```

---

## 9. Signals 信号系统概览

Godot 的信号（Signal）是**观察者模式**的官方实现，用于节点间解耦通信。

```
发射方 (Emitter)  →  emit_signal()  →  接收方 (Listener) 的回调函数
```

### 为什么用信号而不是直接调用函数？

| 直接调用                 | 信号                     |
| ------------------------ | ------------------------ |
| 发射方需要持有接收方引用 | 发射方不需要知道谁在监听 |
| 强耦合                   | 松耦合，易于扩展         |
| 多个接收方需要手动遍历   | 一次 emit 通知所有连接者 |

---

## 10. 内置信号与连接方式

### 10.1 编辑器连接（推荐用于简单情况）

在 Inspector 的 **Node > Signals** 面板中，双击信号 → 选择目标节点 → 自动生成回调函数。

### 10.2 代码连接

```gdscript
# 基本语法
signal_source.some_signal.connect(callable)

# 示例：按钮点击
$Button.pressed.connect(_on_button_pressed)

func _on_button_pressed():
    print("按钮被按下")
```

### 10.3 Lambda 连接

```gdscript
$Timer.timeout.connect(func(): print("时间到！"))
```

### 10.4 断开连接

```gdscript
$Button.pressed.disconnect(_on_button_pressed)

# 检查是否已连接
if $Button.pressed.is_connected(_on_button_pressed):
    $Button.pressed.disconnect(_on_button_pressed)
```

### 10.5 连接标志（Flags）

```gdscript
# CONNECT_ONE_SHOT：触发一次后自动断开
$Area.body_entered.connect(_on_enter, CONNECT_ONE_SHOT)

# CONNECT_DEFERRED：延迟到帧末执行（物理回调中修改场景树时必须用）
$Body.body_entered.connect(_on_enter, CONNECT_DEFERRED)
```

---

## 11. 自定义信号

### 11.1 声明与发射

```gdscript
extends Node

# 无参数信号
signal player_died

# 带参数信号
signal health_changed(old_value: int, new_value: int)

# 带参数发射
func take_damage(amount: int):
    var old_hp = health
    health -= amount
    health_changed.emit(old_hp, health)  # 发射信号

    if health <= 0:
        player_died.emit()
```

### 11.2 接收自定义信号

```gdscript
extends Node

func _ready():
    var player = $Player
    player.health_changed.connect(_on_health_changed)
    player.player_died.connect(_on_player_died)

func _on_health_changed(old_val: int, new_val: int):
    $HUD.update_health_bar(new_val)
    print("血量: %d -> %d" % [old_val, new_val])

func _on_player_died():
    get_tree().change_scene_to_file("res://scenes/game_over.tscn")
```

### 11.3 await 等待信号（协程风格）

```gdscript
func interact_with_door():
    $Door.open()
    await $Door.animation_finished   # 暂停直到信号发出
    print("门已打开，继续执行")
    $Player.move_through_door()
```

---

## 12. 信号与物理的综合实战

### 实战一：拾取物品系统

```gdscript
# pickup_item.gd
extends Area3D

signal item_collected(item_name: String, value: int)

@export var item_name: String = "Gold Coin"
@export var value: int = 10

func _ready():
    body_entered.connect(_on_body_entered)

func _on_body_entered(body: Node3D):
    if body.is_in_group("Player"):
        item_collected.emit(item_name, value)
        queue_free()
```

```gdscript
# player.gd
extends CharacterBody3D

var score: int = 0
signal score_changed(new_score: int)

func _ready():
    # 动态连接场景中所有拾取物
    get_tree().get_nodes_in_group("Pickups").map(func(p):
        p.item_collected.connect(_on_item_collected)
    )

func _on_item_collected(item_name: String, value: int):
    score += value
    print("拾取: ", item_name)
    score_changed.emit(score)
```

---

### 实战二：爆炸范围伤害

```gdscript
# explosion.gd
extends Node3D

signal exploded(position: Vector3, radius: float)

@export var radius: float = 5.0
@export var damage: float = 100.0

func explode():
    # 球形查询
    var space = get_world_3d().direct_space_state
    var params = PhysicsShapeQueryParameters3D.new()
    params.shape = SphereShape3D.new()
    params.shape.radius = radius
    params.transform = global_transform
    params.collision_mask = 0b0011  # 检测层1和层2

    var hits = space.intersect_shape(params)
    for hit in hits:
        var body = hit["collider"]
        if body.has_method("take_damage"):
            # 距离衰减
            var dist = global_position.distance_to(body.global_position)
            var falloff = 1.0 - clamp(dist / radius, 0.0, 1.0)
            body.take_damage(damage * falloff)

    exploded.emit(global_position, radius)

    # 播放效果后销毁
    await $AnimationPlayer.animation_finished
    queue_free()
```

---

### 实战三：平台触发器

```gdscript
# moving_platform_trigger.gd
extends Area3D

@onready var platform: AnimatableBody3D = $"../MovingPlatform"

func _ready():
    body_entered.connect(func(_b): platform.set_process(true))
    body_exited.connect(func(_b):
        if get_overlapping_bodies().is_empty():
            platform.set_process(false)
    )
```

---

## 13. 常见陷阱与最佳实践

### 陷阱 1：在物理回调中直接修改场景树

```gdscript
# 错误：可能导致崩溃
func _on_body_entered(body):
    body.queue_free()  # 有时安全，有时不安全

# 正确：使用 CONNECT_DEFERRED 或 call_deferred
func _on_body_entered(body):
    body.call_deferred("queue_free")

# 或在连接时加 flag
area.body_entered.connect(_on_enter, CONNECT_DEFERRED)
```

### 陷阱 2：重复连接信号

```gdscript
# 错误：每次 _ready 调用都连接，导致多次触发
func _ready():
    $Button.pressed.connect(_on_pressed)  # 如果节点被重新实例化会重复连接

# 正确：连接前检查
func _ready():
    if not $Button.pressed.is_connected(_on_pressed):
        $Button.pressed.connect(_on_pressed)
```

### 陷阱 3：在 \_process 中做物理查询

```gdscript
# 错误：_process 与物理帧不同步，结果不稳定
func _process(delta):
    var result = space.intersect_ray(query)  # 避免

# 正确
func _physics_process(delta):
    var result = space.intersect_ray(query)  # 物理帧中查询
```

### 陷阱 4：CharacterBody 忘记添加重力

```gdscript
# 常见 bug：角色浮空不落地
func _physics_process(delta):
    # 忘记这两行
    if not is_on_floor():
        velocity += get_gravity() * delta
    move_and_slide()
```

### 最佳实践总结

| 实践                          | 说明                                    |
| ----------------------------- | --------------------------------------- |
| 物理逻辑放 `_physics_process` | 确保与物理帧同步                        |
| 用信号解耦节点通信            | 避免父节点持有子节点引用                |
| 用碰撞层过滤不必要检测        | 减少 CPU 开销                           |
| 简单形状优先                  | Box/Sphere > Capsule > Convex > Concave |
| 场景树操作用 `call_deferred`  | 防止物理回调中的崩溃                    |
| `Area` 做触发，`Body` 做碰撞  | 职责清晰，性能更好                      |
| 信号参数用强类型注解          | 提升可读性，便于 IDE 提示               |

---

## 参考资料

- [Godot 官方文档 - Physics introduction](https://docs.godotengine.org/en/stable/tutorials/physics/physics_introduction.html)
- [Godot 官方文档 - Signals](https://docs.godotengine.org/en/stable/getting_started/step_by_step/signals.html)
- [Godot 官方文档 - CharacterBody](https://docs.godotengine.org/en/stable/tutorials/physics/using_character_body_2d.html)
- [GDScript 参考](https://docs.godotengine.org/en/stable/tutorials/scripting/gdscript/gdscript_basics.html)
