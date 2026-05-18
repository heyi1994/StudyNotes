# Godot 常用 2D Node 完整指南

> 适用版本：Godot 4.x

## 1. Node2D — 2D 基类

所有 2D 节点的基类，提供位置、旋转、缩放等变换功能。

### 核心属性

```gdscript
var node = $Node2D

# 局部变换
node.position  = Vector2(100, 200)   # 局部坐标
node.rotation  = deg_to_rad(45)      # 旋转（弧度）
node.scale     = Vector2(2.0, 2.0)   # 缩放

# 全局变换
node.global_position = Vector2(300, 400)
node.global_rotation = deg_to_rad(90)
node.global_scale    = Vector2(1.5, 1.5)

# Z 排序
node.z_index   = 1       # Z 层级，越大越在前
node.z_as_relative = true  # 相对父节点累加
```

### 常用方法

```gdscript
# 朝向目标
node.look_at(target_position)

# 获取朝向向量
var forward = Vector2.RIGHT.rotated(node.rotation)

# 变换空间转换
var local  = node.to_local(global_pos)
var global = node.to_global(local_pos)

# 获取与目标的距离和方向
var dist = node.position.distance_to(target.position)
var dir  = node.position.direction_to(target.position)
```

---

## 2. Sprite2D — 静态精灵

显示一张静态图片。

### 关键属性

```gdscript
var sprite = $Sprite2D

sprite.texture        = load("res://art/player.png")  # 图片资源
sprite.centered       = true       # 以中心为原点
sprite.offset         = Vector2(0, -16)  # 偏移
sprite.flip_h         = false      # 水平翻转
sprite.flip_v         = false      # 垂直翻转

# 图集切割（使用大图中的某一格）
sprite.hframes        = 4          # 横向帧数
sprite.vframes        = 2          # 纵向帧数
sprite.frame          = 3          # 当前帧索引（0开始）

# 等价于手动设置 region
sprite.region_enabled = true
sprite.region_rect    = Rect2(32, 0, 32, 32)

# 调制颜色（乘法混合）
sprite.modulate       = Color(1, 0.5, 0.5, 1)  # 偏红
sprite.self_modulate  = Color(1, 1, 1, 0.5)    # 半透明（只影响自身）
```

### 常用技巧

```gdscript
# 闪烁效果（受伤反馈）
func flash(duration: float):
    var tween = create_tween().set_loops(6)
    tween.tween_property($Sprite2D, "modulate:a", 0.0, duration / 6)
    tween.tween_property($Sprite2D, "modulate:a", 1.0, duration / 6)

# 朝向鼠标翻转
func _process(_delta):
    var mouse_x = get_global_mouse_position().x
    $Sprite2D.flip_h = mouse_x < global_position.x
```

---

## 3. AnimatedSprite2D — 动画精灵

内置帧动画播放，使用 `SpriteFrames` 资源管理动画。

### 基本使用

```gdscript
var anim = $AnimatedSprite2D

# 播放动画
anim.play("run")
anim.play_backwards("run")  # 反向播放
anim.stop()
anim.pause()

# 属性
anim.animation        = "idle"     # 当前动画名
anim.frame            = 0          # 当前帧
anim.speed_scale      = 1.5        # 播放速度倍率
anim.flip_h           = true       # 水平翻转
```

### 信号

```gdscript
func _ready():
    $AnimatedSprite2D.animation_finished.connect(_on_anim_finished)
    $AnimatedSprite2D.frame_changed.connect(_on_frame_changed)

func _on_anim_finished():
    # 一次性动画（如攻击）播完后切回 idle
    $AnimatedSprite2D.play("idle")

func _on_frame_changed():
    # 特定帧触发事件（如第3帧播放脚步音效）
    if $AnimatedSprite2D.frame == 3:
        $StepSound.play()
```

### 用代码创建 SpriteFrames

```gdscript
var frames = SpriteFrames.new()
frames.add_animation("run")
frames.set_animation_speed("run", 12.0)
frames.set_animation_loop("run", true)

for i in range(4):
    var tex = load("res://art/run_%d.png" % i)
    frames.add_frame("run", tex)

$AnimatedSprite2D.sprite_frames = frames
$AnimatedSprite2D.play("run")
```

---

## 4. Camera2D — 摄像机

控制 2D 场景的视角。

### 关键属性

```gdscript
var cam = $Camera2D

cam.enabled           = true
cam.zoom              = Vector2(2, 2)   # 放大 2 倍（值越大场景看起来越小）
cam.offset            = Vector2(0, -50) # 视角偏移
cam.rotation          = deg_to_rad(5)  # 摄像机旋转

# 限制摄像机移动范围
cam.limit_left        = 0
cam.limit_top         = 0
cam.limit_right       = 3200
cam.limit_bottom      = 1800

# 平滑跟随
cam.position_smoothing_enabled = true
cam.position_smoothing_speed   = 5.0
cam.rotation_smoothing_enabled = true
cam.rotation_smoothing_speed   = 3.0

# 拖拽边距（摄像机不立即跟随，到达边缘才移动）
cam.drag_horizontal_enabled = true
cam.drag_vertical_enabled   = true
cam.drag_left_margin        = 0.2
cam.drag_right_margin       = 0.2
cam.drag_top_margin         = 0.2
cam.drag_bottom_margin       = 0.2
```

### 常用技巧

```gdscript
# 屏幕震动
func shake(duration: float, intensity: float):
    var tween = create_tween()
    var end_time = Time.get_ticks_msec() / 1000.0 + duration
    while Time.get_ticks_msec() / 1000.0 < end_time:
        $Camera2D.offset = Vector2(
            randf_range(-intensity, intensity),
            randf_range(-intensity, intensity)
        )
        await get_tree().process_frame
    $Camera2D.offset = Vector2.ZERO

# 平滑缩放
func zoom_to(target_zoom: Vector2, duration: float):
    var tween = create_tween()
    tween.tween_property($Camera2D, "zoom", target_zoom, duration)\
         .set_trans(Tween.TRANS_SINE)
```

---

## 5. 物理体三兄弟

### 5.1 StaticBody2D — 静态体

不会移动的物理体，用于地面、墙壁、平台。

```gdscript
# 基本上只需在编辑器配置，代码极少
var body = $StaticBody2D

# 物理材质（摩擦力、弹性）
var mat = PhysicsMaterial.new()
mat.friction  = 0.5
mat.bounce    = 0.2
body.physics_material_override = mat

# 传送带效果（给角色施加恒定速度）
body.constant_linear_velocity  = Vector2(100, 0)
body.constant_angular_velocity = 0.0
```

### 5.2 CharacterBody2D — 角色体

玩家和 NPC 的首选，提供精确移动控制。

```gdscript
extends CharacterBody2D

const SPEED    = 200.0
const GRAVITY  = 980.0
const JUMP_VEL = -400.0

func _physics_process(delta):
    # 重力
    if not is_on_floor():
        velocity.y += GRAVITY * delta

    # 跳跃
    if Input.is_action_just_pressed("jump") and is_on_floor():
        velocity.y = JUMP_VEL

    # 水平移动
    var dir = Input.get_axis("move_left", "move_right")
    velocity.x = dir * SPEED

    move_and_slide()

# 常用检测方法
func check_state():
    is_on_floor()           # 站在地面
    is_on_ceiling()         # 顶到天花板
    is_on_wall()            # 贴着墙壁
    is_on_wall_only()       # 只贴墙（不在地面）
    get_floor_normal()      # 地面法线向量
    get_floor_angle()       # 地面角度（弧度）
    get_last_slide_collision()  # 最后一次碰撞信息
    get_slide_collision_count() # 本帧碰撞次数
```

### move_and_slide 配置

```gdscript
# 上坡最大角度（超过则视为墙）
up_direction             = Vector2.UP
floor_max_angle          = deg_to_rad(46)

# 停在斜坡上不下滑
floor_stop_on_slope      = true

# 楼梯/小台阶自动爬升高度
floor_snap_length        = 16.0

# 推动 RigidBody2D
motion_mode              = CharacterBody2D.MOTION_MODE_GROUNDED
```

### 5.3 RigidBody2D — 刚体

受物理引擎完全控制，有质量、重力、摩擦力等。

```gdscript
extends RigidBody2D

func _ready():
    mass           = 2.0
    gravity_scale  = 1.0
    linear_damp    = 0.1   # 线性阻尼
    angular_damp   = 1.0   # 角度阻尼

    # 施加力（持续）
    apply_force(Vector2(100, 0))

    # 施加冲量（瞬间）
    apply_impulse(Vector2(0, -500))

    # 在指定位置施加冲量（产生旋转）
    apply_impulse(Vector2(0, -300), Vector2(20, 0))

    # 直接设置速度
    linear_velocity  = Vector2(200, 0)
    angular_velocity = PI

# 物理回调（不要用 _process）
func _integrate_forces(state: PhysicsDirectBodyState2D):
    # state.linear_velocity
    # state.transform
    pass

# 信号
func _ready():
    body_entered.connect(_on_body_entered)

func _on_body_entered(body: Node):
    print("碰到：", body.name)
```

---

## 6. Area2D — 区域检测

不参与物理碰撞，只检测重叠，用于触发器、拾取物、伤害区域等。

```gdscript
extends Area2D

func _ready():
    # 有物体进入/离开
    body_entered.connect(_on_body_entered)
    body_exited.connect(_on_body_exited)

    # 另一个 Area2D 进入/离开
    area_entered.connect(_on_area_entered)
    area_exited.connect(_on_area_exited)

func _on_body_entered(body: Node2D):
    if body.is_in_group("player"):
        print("玩家进入区域")
        body.apply_damage(10)

func _on_body_exited(body: Node2D):
    print("物体离开：", body.name)

# 主动查询当前重叠的所有物体
func get_overlapping():
    var bodies = get_overlapping_bodies()  # Array[Node2D]
    var areas  = get_overlapping_areas()   # Array[Area2D]
    return bodies + areas

# 检测是否有特定物体在内
func has_player() -> bool:
    for body in get_overlapping_bodies():
        if body.is_in_group("player"):
            return true
    return false
```

### 常见用途

```gdscript
# 拾取物
extends Area2D
func _on_body_entered(body):
    if body.is_in_group("player"):
        body.add_item(item_type)
        queue_free()

# 伤害区域（持续）
extends Area2D
var bodies_inside: Array = []

func _ready():
    body_entered.connect(func(b): bodies_inside.append(b))
    body_exited.connect(func(b): bodies_inside.erase(b))

func _process(delta):
    for body in bodies_inside:
        if body.has_method("take_damage"):
            body.take_damage(5 * delta)

# 视野检测
extends Area2D
func can_see_player() -> bool:
    return get_overlapping_bodies().any(func(b): return b.is_in_group("player"))
```

---

## 7. 碰撞形状

### 7.1 CollisionShape2D

配合物理体或 Area2D 使用，定义碰撞区域形状。

```gdscript
# 常用 Shape 类型
var circle  = CircleShape2D.new();   circle.radius = 16.0
var rect    = RectangleShape2D.new(); rect.size = Vector2(32, 48)
var capsule = CapsuleShape2D.new();  capsule.radius = 12; capsule.height = 40
var seg     = SegmentShape2D.new()   # 线段
var ray     = WorldBoundaryShape2D.new()  # 无限平面

$CollisionShape2D.shape    = circle
$CollisionShape2D.disabled = false   # 禁用碰撞（不删除节点）
$CollisionShape2D.debug_color = Color.RED  # 调试颜色

# 运行时禁用碰撞（如无敌帧）
func set_invincible(value: bool):
    $CollisionShape2D.set_deferred("disabled", value)
    # 注意：物理帧内修改碰撞需用 set_deferred，否则报错
```

### 7.2 CollisionPolygon2D

自定义多边形碰撞形状。

```gdscript
var poly = $CollisionPolygon2D

# 设置多边形顶点
poly.polygon = PackedVector2Array([
    Vector2(-20, -30),
    Vector2(20, -30),
    Vector2(30, 0),
    Vector2(20, 30),
    Vector2(-20, 30),
    Vector2(-30, 0),
])

# build_mode 影响碰撞方式
poly.build_mode = CollisionPolygon2D.BUILD_SOLIDS   # 实心
poly.build_mode = CollisionPolygon2D.BUILD_SEGMENTS # 只有边线
```

---

## 8. RayCast2D — 射线检测

从某点向某方向发射射线，检测第一个碰到的物体。

```gdscript
extends Node2D

@onready var ray = $RayCast2D

func _ready():
    ray.enabled        = true
    ray.target_position = Vector2(0, 200)   # 射线终点（局部坐标）
    ray.collision_mask  = 1                  # 只检测 Layer 1

    # 排除自身
    ray.exclude_parent  = true
    ray.collide_with_bodies = true
    ray.collide_with_areas  = false

func _physics_process(_delta):
    if ray.is_colliding():
        var hit_point  = ray.get_collision_point()   # 碰撞点（世界坐标）
        var hit_normal = ray.get_collision_normal()  # 碰撞法线
        var hit_obj    = ray.get_collider()          # 碰到的节点
        var hit_rid    = ray.get_collider_rid()      # 碰到的 RID

        print("碰到：", hit_obj.name, " 在：", hit_point)

# 一次性检测（不需要常驻节点）
func cast_once(from: Vector2, to: Vector2):
    var space = get_world_2d().direct_space_state
    var query = PhysicsRayQueryParameters2D.create(from, to, 1)
    var result = space.intersect_ray(query)
    if result:
        print("命中：", result.collider.name)
    return result
```

---

## 9. ShapeCast2D — 形状扫描

类似 RayCast2D，但用形状代替点，可以检测指定形状沿路径上的碰撞。

```gdscript
@onready var shape_cast = $ShapeCast2D

func _ready():
    var circle = CircleShape2D.new()
    circle.radius = 10.0

    shape_cast.shape          = circle
    shape_cast.target_position = Vector2(0, 100)
    shape_cast.collision_mask  = 1
    shape_cast.max_results     = 5  # 最多检测几个碰撞

func _physics_process(_delta):
    if shape_cast.is_colliding():
        var count = shape_cast.get_collision_count()
        for i in count:
            print(shape_cast.get_collider(i).name)

    # 获取安全移动距离（0.0 ~ 1.0）
    var safe_fraction = shape_cast.get_closest_collision_safe_fraction()
```

---

## 10. VisibleOnScreenNotifier2D — 屏幕可见检测

检测节点是否进入/离开摄像机视野，**不影响节点行为**，只发出信号。

```gdscript
extends Node2D

func _ready():
    $VisibleOnScreenNotifier2D.screen_entered.connect(_on_entered)
    $VisibleOnScreenNotifier2D.screen_exited.connect(_on_exited)

func _on_entered():
    print("进入屏幕，开始处理")
    set_process(true)

func _on_exited():
    print("离开屏幕，停止处理")
    set_process(false)
    queue_free()  # 敌人/子弹移出屏幕时销毁
```

### 调整检测区域

```gdscript
# 默认检测区域是节点的 Rect2
# 可手动设置更大/更小的区域
$VisibleOnScreenNotifier2D.rect = Rect2(-32, -32, 64, 64)
```

### 主动查询

```gdscript
if $VisibleOnScreenNotifier2D.is_on_screen():
    # 当前在屏幕内
    play_animation()
```

---

## 11. VisibleOnScreenEnabler2D — 屏幕可见自动开关

`VisibleOnScreenNotifier2D` 的增强版，**自动**启用/禁用指定节点的处理，无需写信号代码。

```gdscript
@onready var enabler = $VisibleOnScreenEnabler2D

func _ready():
    # 设置控制哪个节点
    enabler.enable_mode = VisibleOnScreenEnabler2D.ENABLE_MODE_INHERIT_PARENT

    # 控制哪些处理被暂停
    enabler.enable_node_path = NodePath("../Enemy")  # 控制同级 Enemy 节点
```

### enable_mode 选项

| 值                           | 说明                                 |
| ---------------------------- | ------------------------------------ |
| `ENABLE_MODE_INHERIT_PARENT` | 控制父节点的 process/physics_process |
| `ENABLE_MODE_ALWAYS`         | 节点始终处于激活状态（不暂停）       |

> **推荐用法：** 将 `VisibleOnScreenEnabler2D` 作为敌人节点的子节点，自动在屏外停止 AI 运算，节省性能。

---

## 12. Timer — 计时器

精确的倒计时工具，到时发出 `timeout` 信号。

```gdscript
# 方式一：节点方式
@onready var timer = $Timer

func _ready():
    timer.wait_time  = 2.0
    timer.one_shot   = true   # true=只触发一次，false=循环
    timer.autostart  = false
    timer.timeout.connect(_on_timeout)
    timer.start()

func _on_timeout():
    print("计时结束")

# 暂停/继续
timer.paused = true
timer.paused = false

# 查询剩余时间
print(timer.time_left)

# 方式二：代码创建（无需节点）
func wait_seconds(sec: float):
    await get_tree().create_timer(sec).timeout
    print("等待完成")

# 实用示例：冷却系统
var skill_cooldown := false

func use_skill():
    if skill_cooldown:
        return
    skill_cooldown = true
    _do_skill()
    await get_tree().create_timer(3.0).timeout
    skill_cooldown = false
```

---

## 13. AudioStreamPlayer2D — 2D 音频

在 2D 世界中播放有位置感的音效（距离越远声音越小）。

```gdscript
@onready var player = $AudioStreamPlayer2D

func _ready():
    player.stream          = load("res://audio/explosion.ogg")
    player.volume_db       = 0.0     # 音量（分贝）
    player.pitch_scale     = 1.0     # 音调（>1 变高）
    player.max_distance    = 500.0   # 最远可听距离
    player.attenuation     = 1.0     # 衰减曲线
    player.bus             = "SFX"   # 音频总线

    player.play()
    player.stop()

# 随机音调（让同一音效有变化感）
func play_random_pitch():
    player.pitch_scale = randf_range(0.9, 1.1)
    player.play()

# 完成播放后自动清理（一次性音效）
func play_oneshot(stream: AudioStream, pos: Vector2):
    var p = AudioStreamPlayer2D.new()
    p.stream   = stream
    p.position = pos
    add_child(p)
    p.play()
    p.finished.connect(p.queue_free)
```

> 若不需要位置感（如背景音乐、UI 音效），用 `AudioStreamPlayer`（无 2D 后缀）。

---

## 14. AnimationPlayer — 动画播放器

关键帧动画系统，可以对**任意节点的任意属性**设置动画。

```gdscript
@onready var anim = $AnimationPlayer

func _ready():
    anim.play("idle")
    anim.play("attack", -1, 1.5)  # 动画名, 混合时间, 速度倍率
    anim.play_backwards("run")
    anim.pause()
    anim.stop()

    # 属性
    anim.current_animation     # 当前动画名
    anim.current_animation_position  # 当前播放位置（秒）
    anim.speed_scale           # 全局速度倍率
    anim.autoplay              # 自动播放的动画名

    # 信号
    anim.animation_finished.connect(_on_anim_finished)
    anim.animation_changed.connect(_on_anim_changed)

func _on_anim_finished(anim_name: String):
    if anim_name == "attack":
        $AnimationPlayer.play("idle")

# 等待动画完成
func do_attack():
    $AnimationPlayer.play("attack")
    await $AnimationPlayer.animation_finished
    # 攻击动画结束后继续执行
    state = State.IDLE
```

---

## 15. AnimationTree — 动画状态机

基于图形的动画混合和状态机，配合 AnimationPlayer 使用。

```gdscript
@onready var tree  = $AnimationTree
@onready var state = $AnimationTree.get("parameters/playback") as AnimationNodeStateMachinePlayback

func _ready():
    tree.active = true

# 切换状态
func set_state_run():
    state.travel("Run")

func set_state_idle():
    state.travel("Idle")

# 设置混合参数
func set_move_blend(direction: float):
    # BlendSpace1D 参数
    tree.set("parameters/BlendSpace1D/blend_position", direction)

    # BlendSpace2D 参数
    tree.set("parameters/BlendSpace2D/blend_position", Vector2(dir_x, dir_y))

# 触发一次性动画（Trigger）
func attack():
    tree.set("parameters/AttackTrigger/request", AnimationNodeOneShot.ONE_SHOT_REQUEST_FIRE)
```

---

## 16. Path2D + PathFollow2D — 路径跟随

让节点沿预设路径移动，常用于巡逻 NPC、弹幕轨迹等。

```gdscript
# Path2D 定义路径曲线（编辑器中绘制贝塞尔曲线）
# PathFollow2D 沿路径移动

@onready var follower = $Path2D/PathFollow2D

var speed := 100.0  # 像素/秒

func _process(delta):
    # progress 是沿路径的像素距离
    follower.progress += speed * delta

    # progress_ratio 是 0.0~1.0 的比例
    # follower.progress_ratio += 0.1 * delta

    # 循环
    follower.loop = true

    # 是否旋转跟随路径方向
    follower.rotates = true

# 代码创建路径
func create_path():
    var path = Path2D.new()
    var curve = Curve2D.new()
    curve.add_point(Vector2(0, 0))
    curve.add_point(Vector2(200, 0))
    curve.add_point(Vector2(200, 200))
    curve.add_point(Vector2(0, 200))
    path.curve = curve
    add_child(path)
```

---

## 17. NavigationAgent2D — 寻路代理

配合 NavigationRegion2D（或 TileMapLayer 的导航层）进行自动寻路。

```gdscript
extends CharacterBody2D

@onready var agent = $NavigationAgent2D

func _ready():
    agent.path_desired_distance    = 4.0   # 到达路径点的判定距离
    agent.target_desired_distance  = 16.0  # 到达目标的判定距离
    agent.max_speed                = 200.0
    agent.navigation_layers        = 1

    # 目标改变时的信号
    agent.navigation_finished.connect(_on_nav_finished)
    agent.target_reached.connect(_on_target_reached)
    agent.waypoint_reached.connect(_on_waypoint_reached)

func move_to(target: Vector2):
    agent.target_position = target

func _physics_process(delta):
    if agent.is_navigation_finished():
        return

    var next_pos  = agent.get_next_path_position()
    var direction = (next_pos - global_position).normalized()

    velocity = direction * 200.0
    move_and_slide()

func _on_nav_finished():
    print("到达目标")

# 避障设置
func setup_avoidance():
    agent.avoidance_enabled = true
    agent.radius            = 16.0
    agent.neighbor_distance = 100.0
    agent.max_neighbors     = 10
```

---

## 18. RemoteTransform2D — 远程变换

将自身的 Transform 同步到另一个节点，常用于摄像机跟随、骨骼挂载等。

```gdscript
# 例：摄像机跟随玩家，但摄像机放在单独的场景中

# 在玩家节点下挂 RemoteTransform2D
@onready var remote = $RemoteTransform2D

func _ready():
    # 指向要同步的目标节点路径
    remote.remote_path         = "/root/World/Camera2D"
    remote.update_position     = true
    remote.update_rotation     = false
    remote.update_scale        = false
    remote.use_global_coordinates = true
```

---

## 19. Marker2D — 标记点

纯粹的位置标记，没有任何功能，用于在场景中标记出生点、巡逻点、炮口位置等。

```gdscript
# Marker2D 本身无属性，只提供 position / rotation

# 常见用途：
# 1. 敌人出生点
var spawn_points = $SpawnPoints.get_children()  # 多个 Marker2D
func get_random_spawn() -> Vector2:
    return spawn_points.pick_random().global_position

# 2. 子弹发射位置
func shoot():
    var bullet = BULLET.instantiate()
    bullet.global_position = $MuzzlePoint.global_position
    bullet.rotation        = $MuzzlePoint.global_rotation
    get_parent().add_child(bullet)

# 3. 巡逻路径点
var patrol_points: Array[Marker2D] = []
var patrol_index := 0

func next_patrol_point() -> Vector2:
    patrol_index = (patrol_index + 1) % patrol_points.size()
    return patrol_points[patrol_index].global_position
```

---

## 20. Line2D — 折线

渲染一条由多个点组成的折线，可设置宽度、颜色、纹理。

```gdscript
@onready var line = $Line2D

func _ready():
    line.width         = 4.0
    line.default_color = Color.RED
    line.begin_cap_mode = Line2D.LINE_CAP_ROUND   # 起点形状
    line.end_cap_mode   = Line2D.LINE_CAP_ROUND   # 终点形状
    line.joint_mode     = Line2D.LINE_JOINT_ROUND # 转折点形状

# 动态绘制轨迹
var trail_points: PackedVector2Array = []

func _process(_delta):
    trail_points.append(global_position)
    if trail_points.size() > 30:
        trail_points.remove_at(0)
    line.points = trail_points

# 清空
line.clear_points()

# 添加/移除点
line.add_point(Vector2(100, 200))
line.remove_point(0)
line.set_point_position(1, Vector2(150, 250))
```

---

## 21. Polygon2D — 多边形

渲染填充多边形，支持纹理、骨骼蒙皮。

```gdscript
@onready var poly = $Polygon2D

func _ready():
    poly.polygon = PackedVector2Array([
        Vector2(-50, -50),
        Vector2(50, -50),
        Vector2(50, 50),
        Vector2(-50, 50),
    ])

    poly.color   = Color(0.2, 0.8, 0.4, 0.8)
    poly.texture = load("res://art/ground.png")

    # UV 坐标（控制纹理映射）
    poly.uv = PackedVector2Array([
        Vector2(0, 0),
        Vector2(1, 0),
        Vector2(1, 1),
        Vector2(0, 1),
    ])

    # 内部孔洞
    poly.internal_vertex_count = 0
```

---

## 22. CPUParticles2D — CPU 粒子

在 CPU 上运算的粒子系统，兼容性好，适合粒子数量少的场合。

```gdscript
@onready var particles = $CPUParticles2D

func _ready():
    particles.emitting        = true
    particles.amount          = 50         # 粒子数量
    particles.lifetime        = 1.5        # 生命周期（秒）
    particles.one_shot        = false      # 是否只发射一次
    particles.explosiveness   = 0.0        # 0=匀速发射 1=同时爆发

    # 方向与散布
    particles.direction       = Vector2(0, -1)  # 发射方向
    particles.spread          = 45.0            # 散布角度

    # 速度
    particles.initial_velocity_min = 50.0
    particles.initial_velocity_max = 150.0

    # 重力
    particles.gravity         = Vector2(0, 98)

    # 缩放
    particles.scale_amount_min = 0.5
    particles.scale_amount_max = 1.5

    # 颜色
    particles.color = Color.YELLOW

# 一次性爆发（如爆炸效果）
func burst(pos: Vector2):
    var p = CPUParticles2D.new()
    p.global_position  = pos
    p.one_shot         = true
    p.explosiveness    = 1.0
    p.amount           = 30
    p.lifetime         = 0.8
    get_parent().add_child(p)
    p.emitting = true
    # 粒子播完后自动销毁
    await get_tree().create_timer(p.lifetime + 0.1).timeout
    p.queue_free()
```

---

## 23. GPUParticles2D — GPU 粒子

在 GPU 上运算，性能更好，支持更多粒子，需要配置 `ParticleProcessMaterial`。

```gdscript
@onready var particles = $GPUParticles2D

func _ready():
    particles.amount      = 500
    particles.lifetime    = 2.0
    particles.emitting    = true

    var mat = ParticleProcessMaterial.new()
    mat.direction                 = Vector3(0, -1, 0)
    mat.spread                    = 30.0
    mat.initial_velocity_min      = 100.0
    mat.initial_velocity_max      = 200.0
    mat.gravity                   = Vector3(0, 98, 0)
    mat.color                     = Color.CYAN

    # 颜色随生命周期变化
    var gradient = Gradient.new()
    gradient.set_color(0, Color.WHITE)
    gradient.set_color(1, Color(1, 1, 1, 0))  # 渐渐透明
    var grad_tex = GradientTexture1D.new()
    grad_tex.gradient = gradient
    mat.color_ramp = grad_tex

    particles.process_material = mat
```

---

## 24. PointLight2D — 点光源

在 2D 场景中投射圆形光照。

```gdscript
@onready var light = $PointLight2D

func _ready():
    light.texture        = load("res://art/light_texture.png")
    light.color          = Color(1.0, 0.8, 0.5)  # 暖黄色
    light.energy         = 1.5
    light.texture_scale  = 2.0        # 光照范围
    light.shadow_enabled = true       # 开启阴影
    light.shadow_color   = Color(0, 0, 0, 0.5)

    # 混合模式
    light.blend_mode = Light2D.BLEND_MODE_ADD  # 叠加（常用）

# 火把闪烁效果
func _process(delta):
    light.energy = 1.0 + sin(Time.get_ticks_msec() * 0.01) * 0.2 \
                       + randf_range(-0.05, 0.05)
```

---

## 25. DirectionalLight2D — 方向光

模拟太阳光等平行光源。

```gdscript
@onready var light = $DirectionalLight2D

func _ready():
    light.color          = Color.WHITE
    light.energy         = 0.8
    light.height         = 0.0         # 光源高度（影响阴影角度）
    light.max_distance   = 10000       # 最大照射距离
    light.shadow_enabled = true
```

---

## 26. LightOccluder2D — 光遮挡

配合 2D 光源使用，定义哪些区域会产生阴影。

```gdscript
@onready var occluder = $LightOccluder2D

func _ready():
    var shape = OccluderPolygon2D.new()
    shape.polygon = PackedVector2Array([
        Vector2(-16, -32),
        Vector2(16, -32),
        Vector2(16, 32),
        Vector2(-16, 32),
    ])
    shape.cull_mode = OccluderPolygon2D.CULL_DISABLED

    occluder.occluder = shape
    occluder.sdf_collision = true  # 参与 SDF 碰撞
```

---

## 27. BackBufferCopy — 后备缓冲

复制屏幕上某区域的像素到纹理，供着色器读取，实现折射、玻璃等效果。

```gdscript
@onready var bbc = $BackBufferCopy

func _ready():
    bbc.copy_mode = BackBufferCopy.COPY_MODE_RECT    # 复制矩形区域
    # 或
    bbc.copy_mode = BackBufferCopy.COPY_MODE_VIEWPORT # 复制整个视口

    bbc.rect = Rect2(-100, -100, 200, 200)  # 复制区域
```

> 通常配合 `ShaderMaterial` 使用，着色器中用 `SCREEN_TEXTURE` 采样复制的内容。

---

## 附录：节点选型速查

```
显示图片           → Sprite2D
帧动画             → AnimatedSprite2D
复杂动画/属性动画  → AnimationPlayer
动画状态机         → AnimationTree
玩家/NPC 移动      → CharacterBody2D
地形/平台          → StaticBody2D
物理模拟（箱子）   → RigidBody2D
触发区域/拾取      → Area2D
摄像机跟随         → Camera2D
射线检测           → RayCast2D
形状扫描           → ShapeCast2D
自动寻路           → NavigationAgent2D
路径跟随           → Path2D + PathFollow2D
屏幕外销毁         → VisibleOnScreenNotifier2D
屏幕外暂停 AI      → VisibleOnScreenEnabler2D
位置标记           → Marker2D
倒计时/冷却        → Timer
定向音效           → AudioStreamPlayer2D
轨迹/连线          → Line2D
填充形状           → Polygon2D
少量粒子           → CPUParticles2D
大量粒子           → GPUParticles2D
动态光照           → PointLight2D / DirectionalLight2D
产生阴影           → LightOccluder2D
屏幕折射效果       → BackBufferCopy + Shader
位置同步           → RemoteTransform2D
```
