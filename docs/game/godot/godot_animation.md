# Godot Animation

## 核心节点概览

| 节点 | 用途 |
|---|---|
| `AnimationPlayer` | 基础动画播放器，关键帧驱动 |
| `AnimationTree` | 状态机 / 混合树，管理复杂动画逻辑 |
| `Tween` | 轻量补间动画，代码驱动 |
| `AnimatedSprite2D` | 帧序列播放，2D 精灵动画 |
| `AnimatedSprite3D` | 帧序列播放，3D 场景中的精灵 |

---

## AnimationPlayer

### 基本结构

```
Node
    └── AnimationPlayer  ← 挂在需要动画的节点下
```

### 编辑器操作

1. 选中 `AnimationPlayer` 节点
2. 底部面板打开 **Animation** 编辑器
3. 点击 **Add Animation** 新建动画
4. 选中要动画的属性，点击钥匙图标插入关键帧

### 常用 API

```gdscript
@onready var anim = $AnimationPlayer

# 播放
anim.play("walk")
anim.play("attack", -1, 2.0)      # 第三个参数为播放速度倍率
anim.play_backwards("walk")       # 倒放

# 停止
anim.stop()
anim.pause()                      # 暂停（保留当前位置）

# 状态查询
anim.is_playing()
anim.current_animation            # 当前动画名称
anim.current_animation_position   # 当前播放位置（秒）
anim.current_animation_length     # 当前动画总时长

# 过渡混合
anim.play("run", 0.3)             # 第二个参数：混合时间（秒）

# 队列播放（上一个结束后自动播放）
anim.queue("idle")

# 跳转到指定时间
anim.seek(1.5)
anim.seek(0.0, true)              # 第二个参数 true = 立即更新
```

### 信号

```gdscript
anim.animation_finished.connect(_on_anim_finished)
anim.animation_changed.connect(_on_anim_changed)
anim.animation_started.connect(_on_anim_started)

func _on_anim_finished(anim_name: StringName):
    if anim_name == "attack":
        anim.play("idle")
```

### 动画库（AnimationLibrary）

Godot 4.x 引入动画库，方便复用：

```gdscript
# 引用动画库中的动画，格式为 "库名/动画名"
anim.play("combat/attack")
anim.play("movement/run")

# 代码添加动画库
var library = AnimationLibrary.new()
anim.add_animation_library("combat", library)
```

---

## AnimationTree

用于管理多个动画之间的**状态切换**和**混合**，适合角色动画系统。

### 基本结构

```
CharacterBody2D
    ├── AnimationPlayer   ← 存储所有动画片段
    └── AnimationTree     ← 管理动画逻辑
```

### 设置步骤

1. 添加 `AnimationTree` 节点
2. 将 `anim_player` 属性指向 `AnimationPlayer`
3. 设置 `tree_root`（选择混合树类型）
4. 勾选 `active = true`

### 状态机（AnimationNodeStateMachine）

```gdscript
@onready var anim_tree = $AnimationTree
@onready var state_machine = anim_tree.get("parameters/playback")

# 切换状态
state_machine.travel("run")      # 自动走过渡路径
state_machine.start("idle")      # 直接跳到指定状态

# 查询
state_machine.get_current_node() # 当前状态名
state_machine.is_playing()
```

### 混合参数

```gdscript
# BlendSpace1D / BlendSpace2D
anim_tree.set("parameters/blend_position", 0.5)        # 1D
anim_tree.set("parameters/blend_position", Vector2(x, y))  # 2D

# BlendTree 中的参数
anim_tree.set("parameters/TimeScale/scale", 1.5)       # 播放速度

# Transition 切换
anim_tree.set("parameters/Transition/transition_request", "run")
```

### 常见混合节点

| 节点 | 说明 |
|---|---|
| `AnimationNodeBlend2` | 两个动画按权重混合 |
| `AnimationNodeBlend3` | 三个动画混合（-1 / 0 / 1） |
| `AnimationNodeBlendSpace1D` | 一维混合空间 |
| `AnimationNodeBlendSpace2D` | 二维混合空间（移动方向） |
| `AnimationNodeStateMachine` | 状态机 |
| `AnimationNodeTimeScale` | 调整播放速度 |
| `AnimationNodeAdd2` | 叠加动画（如受伤叠加在移动上） |

---

## Tween

轻量、灵活的代码补间，无需 AnimationPlayer，适合 UI 动画、一次性效果。

### 创建方式

```gdscript
# 方式一：节点绑定（节点销毁时自动停止）
var tween = create_tween()

# 方式二：独立 Tween（手动管理生命周期）
var tween = get_tree().create_tween()
```

### 基本用法

```gdscript
# 移动到目标位置，耗时 1 秒
tween.tween_property($Sprite, "position", Vector2(200, 100), 1.0)

# 链式调用（顺序执行）
tween.tween_property($Sprite, "position", Vector2(200, 0), 0.5)
tween.tween_property($Sprite, "modulate:a", 0.0, 0.3)  # 渐隐

# 并行执行
tween.set_parallel(true)
tween.tween_property($Sprite, "position", Vector2(200, 0), 0.5)
tween.tween_property($Sprite, "scale", Vector2(2, 2), 0.5)
```

### 缓动曲线

```gdscript
tween.tween_property(...).set_trans(Tween.TRANS_BOUNCE)
tween.tween_property(...).set_ease(Tween.EASE_OUT)

# 常用 TRANS 类型
# TRANS_LINEAR   线性
# TRANS_SINE     正弦
# TRANS_QUINT    五次方
# TRANS_BOUNCE   弹跳
# TRANS_ELASTIC  弹性
# TRANS_BACK     回弹

# EASE 类型
# EASE_IN        慢进快出
# EASE_OUT       快进慢出
# EASE_IN_OUT    两端慢
```

### 其他方法

```gdscript
# 延迟
tween.tween_interval(0.5)          # 等待 0.5 秒

# 回调
tween.tween_callback(func(): print("done"))
tween.tween_callback(_on_tween_done)

# 自定义插值（每帧调用）
tween.tween_method(_my_func, 0.0, 1.0, 1.0)

# 控制
tween.pause()
tween.resume()
tween.kill()                       # 立即销毁

# 循环
tween.set_loops(3)                 # 循环 3 次
tween.set_loops()                  # 无限循环
```

---

## AnimatedSprite

### AnimatedSprite2D

帧序列动画，适合像素风格游戏。

```
AnimatedSprite2D
    └── SpriteFrames  ← 动画帧集合（编辑器中配置）
```

```gdscript
@onready var sprite = $AnimatedSprite2D

sprite.play("walk")
sprite.play("idle", true)          # 第二个参数：是否从头播放
sprite.stop()
sprite.pause()

sprite.animation                   # 当前动画名
sprite.frame                       # 当前帧索引
sprite.speed_scale                 # 播放速度倍率
sprite.flip_h                      # 水平翻转
sprite.flip_v                      # 垂直翻转

# 信号
sprite.animation_finished.connect(_on_finished)
sprite.frame_changed.connect(_on_frame_changed)
```

### SpriteFrames 代码创建

```gdscript
var frames = SpriteFrames.new()
frames.add_animation("walk")
frames.set_animation_speed("walk", 10)   # FPS
frames.set_animation_loop("walk", true)

for i in range(8):
    var tex = load("res://frames/walk_%d.png" % i)
    frames.add_frame("walk", tex)

$AnimatedSprite2D.sprite_frames = frames
```

---

## 动画轨道类型

在 AnimationPlayer 编辑器中，可添加以下轨道类型：

| 轨道类型 | 说明 |
|---|---|
| **Property** | 对节点任意属性插值（位置、颜色、大小等） |
| **Call Method** | 在指定时间点调用方法 |
| **Bezier Curve** | 贝塞尔曲线控制的属性插值 |
| **Audio Playback** | 在指定时间播放音效 |
| **Animation Playback** | 嵌套播放其他 AnimationPlayer 的动画 |
| **Blend Shape** | 控制 3D 模型的形变 |

### 代码添加关键帧

```gdscript
var animation = Animation.new()

# 添加位置轨道
var track_idx = animation.add_track(Animation.TYPE_VALUE)
animation.track_set_path(track_idx, "Sprite2D:position")
animation.track_insert_key(track_idx, 0.0, Vector2(0, 0))    # 时间, 值
animation.track_insert_key(track_idx, 1.0, Vector2(200, 0))

animation.length = 1.0
animation.loop_mode = Animation.LOOP_LINEAR

$AnimationPlayer.add_animation("move", animation)
$AnimationPlayer.play("move")
```

---

## 动画混合与过渡

### AnimationPlayer 混合

```gdscript
# play 的第二个参数是混合时间
anim.play("run", 0.2)     # 从当前动画用 0.2 秒过渡到 run
```

### AnimationTree 过渡条件

在状态机编辑器中，连线上可设置：

- **条件（Condition）**：布尔值触发
- **自动过渡**：动画结束后自动跳转

```gdscript
# 设置条件触发
anim_tree.set("parameters/conditions/is_running", true)
anim_tree.set("parameters/conditions/is_running", false)
```

---

## 常见场景示例

### 角色移动动画

```gdscript
extends CharacterBody2D

@onready var anim = $AnimationPlayer

func _physics_process(delta):
    var direction = Input.get_axis("ui_left", "ui_right")

    if direction != 0:
        velocity.x = direction * 200
        $Sprite2D.flip_h = direction < 0
        if anim.current_animation != "walk":
            anim.play("walk")
    else:
        velocity.x = move_toward(velocity.x, 0, 200)
        if anim.current_animation != "idle":
            anim.play("idle")

    move_and_slide()
```

### UI 弹出动画（Tween）

```gdscript
func show_panel():
    $Panel.visible = true
    $Panel.scale = Vector2.ZERO
    var tween = create_tween()
    tween.tween_property($Panel, "scale", Vector2.ONE, 0.3)\
         .set_trans(Tween.TRANS_BACK)\
         .set_ease(Tween.EASE_OUT)

func hide_panel():
    var tween = create_tween()
    tween.tween_property($Panel, "scale", Vector2.ZERO, 0.2)\
         .set_trans(Tween.TRANS_QUAD)\
         .set_ease(Tween.EASE_IN)
    tween.tween_callback(func(): $Panel.visible = false)
```

### 攻击动画 + 判定帧

```gdscript
@onready var anim = $AnimationPlayer

func attack():
    anim.play("attack")

# 在 AnimationPlayer 的 Call Method 轨道中，
# 在判定帧时间点调用此方法
func _enable_hitbox():
    $HitBox.monitoring = true

func _disable_hitbox():
    $HitBox.monitoring = false
```

### 状态机角色动画

```gdscript
extends CharacterBody2D

@onready var anim_tree = $AnimationTree
@onready var state_machine = anim_tree.get("parameters/playback")

func _physics_process(delta):
    var on_floor = is_on_floor()
    var moving = abs(velocity.x) > 10

    if not on_floor:
        state_machine.travel("jump")
    elif moving:
        state_machine.travel("run")
    else:
        state_machine.travel("idle")
```

---

## 快速选型

| 场景 | 推荐方案 |
|---|---|
| 角色帧序列（像素游戏） | `AnimatedSprite2D` |
| 属性关键帧动画 | `AnimationPlayer` |
| 角色多状态动画切换 | `AnimationTree` 状态机 |
| UI 过渡 / 一次性效果 | `Tween` |
| 复杂角色混合（跑+射击） | `AnimationTree` 混合树 |
| 程序化动画 | `Tween` 或代码直接操作属性 |
