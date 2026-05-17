# Godot 4 Input

## 核心概念

### 两种输入哲学

Godot 的输入系统有两种获取方式，理解差别很重要：

| 方式                | 时机           | 适合场景                          |
| ------------------- | -------------- | --------------------------------- |
| **轮询（Polling）** | 每帧主动询问   | 持续性操作（移动、瞄准）          |
| **事件（Event）**   | 发生时被动响应 | 一次性触发（跳跃、开枪、UI 点击） |

```gdscript
# 轮询 — 在 _process 里问"现在按着吗？"
func _process(delta):
    if Input.is_action_pressed("move_right"):
        position.x += speed * delta

# 事件 — 在 _input 里等"有没有新事件？"
func _input(event):
    if event.is_action_pressed("jump"):
        jump()
```

---

### InputEvent 家族

所有输入事件都继承自 `InputEvent`，常用子类：

```
InputEvent
├── InputEventKey              # 键盘按键
├── InputEventMouseButton      # 鼠标按键
├── InputEventMouseMotion      # 鼠标移动
├── InputEventJoypadButton     # 手柄按键
├── InputEventJoypadMotion     # 手柄摇杆/扳机
├── InputEventScreenTouch      # 触摸开始/结束
├── InputEventScreenDrag       # 触摸拖动
└── InputEventAction           # 虚拟动作事件
```

---

## 三种处理方式

### \_input(event) — 最先响应

```gdscript
func _input(event: InputEvent) -> void:
    # 最先收到事件，所有节点都能收到
    if event is InputEventKey:
        print("按键事件：", event.keycode)

    # 标记事件已处理，阻止继续传播
    get_viewport().set_input_as_handled()
```

**特点：**

- 比 `_unhandled_input` 先触发
- GUI 控件处理前就会收到
- 适合全局快捷键（如截图、暂停）

---

### \_unhandled_input(event) — GUI 处理后

```gdscript
func _unhandled_input(event: InputEvent) -> void:
    # GUI 已处理过的事件不会到这里
    # 适合游戏逻辑，防止点 UI 时触发游戏操作
    if event.is_action_pressed("shoot"):
        fire()
```

**特点：**

- GUI 元素（Button、LineEdit 等）消费事件后，这里收不到
- **游戏逻辑首选**，避免点按钮时误触发游戏操作

---

### \_gui_input(event) — 仅 Control 节点

```gdscript
# 只在继承 Control 的节点上有效
func _gui_input(event: InputEvent) -> void:
    if event is InputEventMouseButton:
        if event.button_index == MOUSE_BUTTON_LEFT and event.pressed:
            print("控件被点击")
```

---

### 三者优先级

```
事件发生
    ↓
_input()           ← 全局最先
    ↓
Control._gui_input() ← GUI 控件处理
    ↓
_unhandled_input() ← 游戏逻辑（GUI 没消费才到这）
```

---

## InputMap

`InputMap` 是动作映射系统，将具体按键抽象为"动作名称"，实现按键可配置。

### 在编辑器中配置

**项目 → 项目设置 → 输入映射**

添加动作名（如 `jump`），然后为它绑定按键/按钮/摇杆。

---

### 运行时操作 InputMap

```gdscript
# 检查动作是否存在
InputMap.has_action("jump")

# 运行时添加自定义动作
InputMap.add_action("custom_action", 0.5)  # 第二参数是死区

# 为动作绑定按键
var key_event = InputEventKey.new()
key_event.keycode = KEY_SPACE
InputMap.action_add_event("custom_action", key_event)

# 清除动作的所有绑定
InputMap.action_erase_events("jump")

# 删除整个动作
InputMap.erase_action("custom_action")

# 获取动作的所有绑定
var events = InputMap.action_get_events("jump")
for e in events:
    print(e)

# 恢复默认设置
InputMap.load_from_project_settings()
```

---

### 保存自定义按键绑定

```gdscript
# 保存到文件
func save_keybindings() -> void:
    var config = ConfigFile.new()
    for action in ["jump", "shoot", "move_left", "move_right"]:
        var events = InputMap.action_get_events(action)
        # 只保存键盘事件
        for event in events:
            if event is InputEventKey:
                config.set_value("keybindings", action, event.keycode)
    config.save("user://keybindings.cfg")

# 从文件加载
func load_keybindings() -> void:
    var config = ConfigFile.new()
    if config.load("user://keybindings.cfg") != OK:
        return
    for action in config.get_section_keys("keybindings"):
        var keycode = config.get_value("keybindings", action)
        var event = InputEventKey.new()
        event.keycode = keycode
        InputMap.action_erase_events(action)
        InputMap.action_add_event(action, event)
```

---

## 键盘输入

### Input 单例轮询

```gdscript
func _process(delta: float) -> void:
    # 动作（推荐，与具体按键解耦）
    if Input.is_action_pressed("move_left"):   # 持续按住
        position.x -= speed * delta
    if Input.is_action_just_pressed("jump"):   # 仅按下瞬间
        jump()
    if Input.is_action_just_released("shoot"): # 仅松开瞬间
        release_charge()

    # 获取动作强度（0.0 ~ 1.0，键盘通常是 0 或 1）
    var strength = Input.get_action_strength("accelerate")

    # 组合两个相反方向为 -1 ~ 1 的轴
    var h = Input.get_axis("move_left", "move_right")  # 负左正右
    var v = Input.get_axis("move_up", "move_down")

    # 获取 2D 向量（自动归一化，防止斜向更快）
    var dir = Input.get_vector("move_left", "move_right", "move_up", "move_down")
    velocity = dir * speed
```

---

### \_input 事件处理

```gdscript
func _input(event: InputEvent) -> void:
    if event is InputEventKey:
        var key: InputEventKey = event

        # 基本属性
        print(key.keycode)          # 物理键码（KEY_A、KEY_SPACE 等）
        print(key.unicode)          # Unicode 字符码（考虑输入法）
        print(key.pressed)          # true=按下, false=松开
        print(key.echo)             # true=长按重复触发
        print(key.physical_keycode) # 物理位置键码（不受布局影响）

        # 过滤重复触发
        if key.echo:
            return

        if key.pressed and key.keycode == KEY_ESCAPE:
            get_tree().quit()
```

---

### 常用 KeyCode

```gdscript
# 字母
KEY_A ... KEY_Z

# 数字
KEY_0 ... KEY_9
KEY_KP_0 ... KEY_KP_9    # 小键盘

# 功能键
KEY_F1 ... KEY_F12
KEY_ESCAPE
KEY_ENTER
KEY_SPACE
KEY_TAB
KEY_BACKSPACE
KEY_DELETE
KEY_INSERT

# 方向键
KEY_LEFT, KEY_RIGHT, KEY_UP, KEY_DOWN

# 修饰键
KEY_SHIFT, KEY_CTRL, KEY_ALT, KEY_META  # Meta = Win/Cmd

# 特殊
KEY_HOME, KEY_END, KEY_PAGEUP, KEY_PAGEDOWN
```

---

## 鼠标输入

### 鼠标按键事件

```gdscript
func _input(event: InputEvent) -> void:
    if event is InputEventMouseButton:
        var mb: InputEventMouseButton = event

        print(mb.position)          # 相对于视口的位置
        print(mb.global_position)   # 全局位置
        print(mb.pressed)           # 按下还是松开
        print(mb.double_click)      # 是否双击
        print(mb.button_index)      # 哪个按键

        match mb.button_index:
            MOUSE_BUTTON_LEFT:
                if mb.pressed:
                    start_drag(mb.position)
                else:
                    end_drag()

            MOUSE_BUTTON_RIGHT:
                if mb.pressed:
                    show_context_menu(mb.position)

            MOUSE_BUTTON_MIDDLE:
                if mb.pressed:
                    reset_camera()

            MOUSE_BUTTON_WHEEL_UP:
                zoom_in()

            MOUSE_BUTTON_WHEEL_DOWN:
                zoom_out()

            MOUSE_BUTTON_XBUTTON1:  # 侧键后退
                go_back()

            MOUSE_BUTTON_XBUTTON2:  # 侧键前进
                go_forward()
```

---

### 鼠标移动事件

```gdscript
func _input(event: InputEvent) -> void:
    if event is InputEventMouseMotion:
        var mm: InputEventMouseMotion = event

        print(mm.position)          # 当前位置（视口坐标）
        print(mm.relative)          # 相对上一帧的位移
        print(mm.velocity)          # 移动速度（像素/秒）
        print(mm.pressure)          # 触摸板压力（0.0~1.0）
        print(mm.tilt)              # 触控笔倾斜（Vector2）

# 第一人称视角：用 relative 旋转摄像机
func _input(event: InputEvent) -> void:
    if event is InputEventMouseMotion and Input.mouse_mode == Input.MOUSE_MODE_CAPTURED:
        rotate_y(-event.relative.x * sensitivity)
        camera.rotate_x(-event.relative.y * sensitivity)
        camera.rotation.x = clamp(camera.rotation.x, -PI/2, PI/2)
```

---

### 轮询鼠标状态

```gdscript
func _process(_delta: float) -> void:
    # 当前位置
    var pos = get_viewport().get_mouse_position()

    # 按键状态（位掩码）
    var buttons = Input.get_mouse_button_mask()
    if buttons & MOUSE_BUTTON_MASK_LEFT:
        print("左键按住")
    if buttons & MOUSE_BUTTON_MASK_RIGHT:
        print("右键按住")
```

---

## 手柄输入

### 手柄连接检测

```gdscript
func _ready() -> void:
    Input.joy_connection_changed.connect(_on_joy_connection_changed)

    # 获取已连接的手柄
    var gamepads = Input.get_connected_joypads()
    for id in gamepads:
        print("手柄 %d: %s" % [id, Input.get_joy_name(id)])

func _on_joy_connection_changed(device_id: int, connected: bool) -> void:
    if connected:
        print("手柄 %d 已连接: %s" % [device_id, Input.get_joy_name(device_id)])
    else:
        print("手柄 %d 已断开" % device_id)
```

---

### 手柄按键轮询

```gdscript
func _process(delta: float) -> void:
    # 通过 InputMap 动作（推荐，自动支持所有设备）
    if Input.is_action_pressed("jump"):
        # 在 InputMap 里把 jump 绑定到 JOY_BUTTON_A
        pass

    # 直接查询特定手柄特定按钮
    var device = 0  # 第一个手柄
    if Input.is_joy_button_pressed(device, JOY_BUTTON_A):
        jump()

    # 摇杆轴（-1.0 ~ 1.0）
    var left_x = Input.get_joy_axis(device, JOY_AXIS_LEFT_X)
    var left_y = Input.get_joy_axis(device, JOY_AXIS_LEFT_Y)

    # 扳机（0.0 ~ 1.0）
    var right_trigger = Input.get_joy_axis(device, JOY_AXIS_TRIGGER_RIGHT)

    # 推荐：用 get_vector 自动处理死区
    var move = Input.get_vector(
        "move_left", "move_right", "move_up", "move_down"
    )
```

---

### 常用手柄常量

```gdscript
# 按键
JOY_BUTTON_A          # 下键（Xbox A / PS X）
JOY_BUTTON_B          # 右键（Xbox B / PS O）
JOY_BUTTON_X          # 左键（Xbox X / PS □）
JOY_BUTTON_Y          # 上键（Xbox Y / PS △）
JOY_BUTTON_LEFT_SHOULDER   # LB / L1
JOY_BUTTON_RIGHT_SHOULDER  # RB / R1
JOY_BUTTON_LEFT_STICK      # L3（按下左摇杆）
JOY_BUTTON_RIGHT_STICK     # R3（按下右摇杆）
JOY_BUTTON_START
JOY_BUTTON_BACK       # Select / Share
JOY_BUTTON_DPAD_UP
JOY_BUTTON_DPAD_DOWN
JOY_BUTTON_DPAD_LEFT
JOY_BUTTON_DPAD_RIGHT

# 轴
JOY_AXIS_LEFT_X       # 左摇杆水平
JOY_AXIS_LEFT_Y       # 左摇杆垂直
JOY_AXIS_RIGHT_X      # 右摇杆水平
JOY_AXIS_RIGHT_Y      # 右摇杆垂直
JOY_AXIS_TRIGGER_LEFT  # LT / L2 (0.0~1.0)
JOY_AXIS_TRIGGER_RIGHT # RT / R2 (0.0~1.0)
```

---

### 手柄震动

```gdscript
# 震动（device, 弱马达强度, 强马达强度, 持续时间秒）
Input.start_joy_vibration(0, 0.0, 1.0, 0.5)  # 强震动 0.5 秒
Input.start_joy_vibration(0, 0.5, 0.5, 0.2)  # 双马达 0.2 秒
Input.stop_joy_vibration(0)                    # 立即停止
```

---

### 死区设置

```gdscript
# 在 InputMap 中为动作设置死区（0.0~1.0）
InputMap.add_action("move_right", 0.2)  # 死区 20%

# 或在项目设置里全局配置
# 手动处理死区：
func apply_deadzone(value: float, deadzone: float = 0.2) -> float:
    if abs(value) < deadzone:
        return 0.0
    return sign(value) * (abs(value) - deadzone) / (1.0 - deadzone)
```

---

## 触摸屏输入

### 触摸事件

```gdscript
func _input(event: InputEvent) -> void:
    # 触摸开始/结束
    if event is InputEventScreenTouch:
        var touch: InputEventScreenTouch = event
        print("手指 ID:", touch.index)        # 多点触摸的手指编号
        print("位置:", touch.position)
        print("按下:", touch.pressed)         # true=按下，false=抬起

        if touch.pressed:
            fingers[touch.index] = touch.position
        else:
            fingers.erase(touch.index)

    # 触摸拖动
    if event is InputEventScreenDrag:
        var drag: InputEventScreenDrag = event
        print("手指 ID:", drag.index)
        print("当前位置:", drag.position)
        print("相对位移:", drag.relative)
        print("速度:", drag.velocity)
```

---

### 多点触控（捏合缩放示例）

```gdscript
var _touch_points: Dictionary = {}

func _input(event: InputEvent) -> void:
    if event is InputEventScreenTouch:
        if event.pressed:
            _touch_points[event.index] = event.position
        else:
            _touch_points.erase(event.index)

    if event is InputEventScreenDrag:
        _touch_points[event.index] = event.position

        if _touch_points.size() == 2:
            _handle_pinch()

func _handle_pinch() -> void:
    var points = _touch_points.values()
    var dist = points[0].distance_to(points[1])
    # 用当前距离与上一帧距离的比值来缩放
    # ...
```

---

### 模拟触摸（在 PC 上测试）

**项目设置 → 输入设备 → 触摸 → 模拟鼠标的触控**

或在代码中：

```gdscript
# 项目设置里开启：将触控模拟为鼠标，或将鼠标模拟为触控
ProjectSettings.set_setting("input_devices/pointing/emulate_touch_from_mouse", true)
```

---

## 输入修饰键

### 检测组合键

```gdscript
func _input(event: InputEvent) -> void:
    if event is InputEventKey and event.pressed and not event.echo:
        # 方式一：检查修饰键属性
        if event.ctrl_pressed and event.keycode == KEY_S:
            save()
        if event.ctrl_pressed and event.shift_pressed and event.keycode == KEY_Z:
            redo()

        # 方式二：检查 key_label（考虑键盘布局）
        if event.is_match(InputEventKey.new()):
            pass

    # 随时查询修饰键状态
    if Input.is_key_pressed(KEY_SHIFT):
        print("Shift 按住中")
```

---

### 修饰键属性

```gdscript
event.shift_pressed   # Shift
event.ctrl_pressed    # Ctrl
event.alt_pressed     # Alt
event.meta_pressed    # Win / Cmd（macOS）

# 跨平台快捷键（自动判断 Ctrl/Cmd）
event.is_command_or_control_autoremap()
```

---

## 鼠标模式与光标

### 鼠标模式

```gdscript
# 四种模式
Input.mouse_mode = Input.MOUSE_MODE_VISIBLE        # 默认，可见
Input.mouse_mode = Input.MOUSE_MODE_HIDDEN         # 隐藏，仍可移动
Input.mouse_mode = Input.MOUSE_MODE_CAPTURED       # 锁定+隐藏（FPS）
Input.mouse_mode = Input.MOUSE_MODE_CONFINED       # 限制在窗口内
Input.mouse_mode = Input.MOUSE_MODE_CONFINED_HIDDEN # 限制+隐藏

# FPS 游戏标准做法
func _ready() -> void:
    Input.mouse_mode = Input.MOUSE_MODE_CAPTURED

func _input(event: InputEvent) -> void:
    if event.is_action_pressed("ui_cancel"):
        Input.mouse_mode = Input.MOUSE_MODE_VISIBLE
```

---

### 自定义光标

```gdscript
# 设置全局光标
var cursor = load("res://cursor.png")
Input.set_custom_mouse_cursor(cursor)

# 带热点（点击位置偏移）
Input.set_custom_mouse_cursor(cursor, Input.CURSOR_ARROW, Vector2(8, 8))

# 不同状态不同光标
Input.set_custom_mouse_cursor(crosshair, Input.CURSOR_CROSS)
Input.set_custom_mouse_cursor(pointer, Input.CURSOR_POINTING_HAND)

# 恢复系统光标
Input.set_custom_mouse_cursor(null)
```

---

### Control 节点光标

```gdscript
# 鼠标悬停在 Control 上时自动切换光标
$Button.mouse_default_cursor_shape = Control.CURSOR_POINTING_HAND
$ResizeHandle.mouse_default_cursor_shape = Control.CURSOR_HSIZE
```

---

## 实战案例

### 案例一：2D 角色控制器

```gdscript
extends CharacterBody2D

const SPEED = 200.0
const JUMP_VELOCITY = -400.0
const GRAVITY = 980.0

func _physics_process(delta: float) -> void:
    # 重力
    if not is_on_floor():
        velocity.y += GRAVITY * delta

    # 跳跃（事件在 _unhandled_input 处理更好，这里演示 just_pressed）
    if Input.is_action_just_pressed("jump") and is_on_floor():
        velocity.y = JUMP_VELOCITY

    # 水平移动
    var dir = Input.get_axis("move_left", "move_right")
    if dir != 0:
        velocity.x = dir * SPEED
    else:
        velocity.x = move_toward(velocity.x, 0, SPEED * delta * 10)

    move_and_slide()
```

---

### 案例二：第一人称视角

```gdscript
extends CharacterBody3D

@export var move_speed = 5.0
@export var mouse_sensitivity = 0.003
@export var jump_velocity = 5.0

@onready var camera = $Camera3D

const GRAVITY = -9.8

func _ready() -> void:
    Input.mouse_mode = Input.MOUSE_MODE_CAPTURED

func _unhandled_input(event: InputEvent) -> void:
    # ESC 释放鼠标
    if event.is_action_pressed("ui_cancel"):
        Input.mouse_mode = Input.MOUSE_MODE_VISIBLE

    # 点击重新捕获
    if event is InputEventMouseButton and event.pressed:
        Input.mouse_mode = Input.MOUSE_MODE_CAPTURED

    # 鼠标旋转
    if event is InputEventMouseMotion and Input.mouse_mode == Input.MOUSE_MODE_CAPTURED:
        rotate_y(-event.relative.x * mouse_sensitivity)
        camera.rotate_x(-event.relative.y * mouse_sensitivity)
        camera.rotation.x = clamp(camera.rotation.x, -PI / 2.2, PI / 2.2)

func _physics_process(delta: float) -> void:
    if not is_on_floor():
        velocity.y += GRAVITY * delta

    if Input.is_action_just_pressed("jump") and is_on_floor():
        velocity.y = jump_velocity

    var input_dir = Input.get_vector("move_left", "move_right", "move_forward", "move_back")
    var direction = (transform.basis * Vector3(input_dir.x, 0, input_dir.y)).normalized()

    if direction:
        velocity.x = direction.x * move_speed
        velocity.z = direction.z * move_speed
    else:
        velocity.x = move_toward(velocity.x, 0, move_speed)
        velocity.z = move_toward(velocity.z, 0, move_speed)

    move_and_slide()
```

---

### 案例三：输入缓冲（土狼时间 + 跳跃预输入）

```gdscript
extends CharacterBody2D

const SPEED = 200.0
const JUMP_VELOCITY = -500.0
const GRAVITY = 1200.0

# 土狼时间：离开平台后仍可跳跃的时间窗口
var coyote_timer := 0.0
const COYOTE_TIME = 0.12

# 跳跃缓冲：提前按跳跃，落地后自动跳
var jump_buffer_timer := 0.0
const JUMP_BUFFER_TIME = 0.15

var was_on_floor := false

func _unhandled_input(event: InputEvent) -> void:
    if event.is_action_pressed("jump"):
        jump_buffer_timer = JUMP_BUFFER_TIME

func _physics_process(delta: float) -> void:
    # 重力
    if not is_on_floor():
        velocity.y += GRAVITY * delta

    # 土狼时间计时
    if was_on_floor and not is_on_floor():
        coyote_timer = COYOTE_TIME
    coyote_timer = max(0.0, coyote_timer - delta)

    # 跳跃缓冲计时
    jump_buffer_timer = max(0.0, jump_buffer_timer - delta)

    # 可以跳跃的条件：(在地上 OR 土狼时间内) AND 缓冲队列有跳跃
    var can_jump = is_on_floor() or coyote_timer > 0.0
    if can_jump and jump_buffer_timer > 0.0:
        velocity.y = JUMP_VELOCITY
        coyote_timer = 0.0
        jump_buffer_timer = 0.0

    # 松开跳跃键时截断上升（可变高度跳跃）
    if Input.is_action_just_released("jump") and velocity.y < 0:
        velocity.y *= 0.5

    # 水平移动
    var dir = Input.get_axis("move_left", "move_right")
    velocity.x = dir * SPEED

    was_on_floor = is_on_floor()
    move_and_slide()
```

---

### 案例四：点击移动（RTS 风格）

```gdscript
extends CharacterBody2D

var target_position: Vector2
var moving := false

func _unhandled_input(event: InputEvent) -> void:
    if event is InputEventMouseButton:
        if event.button_index == MOUSE_BUTTON_RIGHT and event.pressed:
            # 世界坐标 = 视口坐标转换
            target_position = get_global_mouse_position()
            moving = true

func _physics_process(delta: float) -> void:
    if not moving:
        return

    var direction = (target_position - global_position)
    if direction.length() < 5.0:
        moving = false
        velocity = Vector2.ZERO
    else:
        velocity = direction.normalized() * 200.0

    move_and_slide()
```

---

### 案例五：可重绑定按键 UI

```gdscript
extends Control

var listening_for: String = ""
var listening_button: Button = null

func start_listening(action: String, button: Button) -> void:
    listening_for = action
    listening_button = button
    button.text = "按下任意键..."
    set_process_unhandled_input(true)

func _unhandled_input(event: InputEvent) -> void:
    if listening_for == "":
        return

    # 只接受键盘和手柄按键
    if not (event is InputEventKey or event is InputEventJoypadButton):
        return

    # ESC 取消
    if event is InputEventKey and event.keycode == KEY_ESCAPE:
        listening_button.text = get_action_display(listening_for)
        listening_for = ""
        return

    # 绑定新按键
    InputMap.action_erase_events(listening_for)
    InputMap.action_add_event(listening_for, event)
    listening_button.text = event.as_text()
    listening_for = ""
    get_viewport().set_input_as_handled()

func get_action_display(action: String) -> String:
    var events = InputMap.action_get_events(action)
    if events.is_empty():
        return "未绑定"
    return events[0].as_text()
```

---

## 常见问题

### Q: \_input 和 \_unhandled_input 该用哪个？

```
游戏逻辑 → _unhandled_input（防止 UI 点击穿透到游戏）
全局快捷键 → _input（总是响应，不被 UI 拦截）
UI 控件内部 → _gui_input
```

---

### Q: 如何防止斜向移动更快？

```gdscript
# 错误：直接相加，斜向速度是 √2 倍
velocity.x = h * speed
velocity.y = v * speed

# 正确方式一：归一化
var dir = Vector2(h, v).normalized()
velocity = dir * speed

# 正确方式二：直接用 get_vector（已自动归一化）
var dir = Input.get_vector("left", "right", "up", "down")
velocity = dir * speed
```

---

### Q: 手柄和键鼠如何同时支持？

```gdscript
# 在 InputMap 中为同一动作绑定多个输入源
# jump → Space 键 + 手柄 A 键
# 代码无需改动，Input.is_action_pressed("jump") 自动识别

# 检测当前使用的输入设备
func _input(event: InputEvent) -> void:
    if event is InputEventKey or event is InputEventMouseButton:
        current_input_device = "keyboard_mouse"
    elif event is InputEventJoypadButton or event is InputEventJoypadMotion:
        current_input_device = "gamepad"
        # 根据设备显示不同的提示图标
```

---

### Q: 如何在 \_process 中处理一次性跳跃？

```gdscript
# 错误：_process 里 just_pressed 会在某些帧错过
func _process(_delta):
    if Input.is_action_just_pressed("jump"):  # 可能丢帧
        jump()

# 正确：一次性输入放到 _unhandled_input
func _unhandled_input(event: InputEvent) -> void:
    if event.is_action_pressed("jump"):
        jump()
```

---

## 速查卡

```gdscript
# ── 轮询 ──────────────────────────────────────────
Input.is_action_pressed("x")        # 按住
Input.is_action_just_pressed("x")   # 刚按下
Input.is_action_just_released("x")  # 刚松开
Input.get_action_strength("x")      # 强度 0~1
Input.get_axis("neg", "pos")        # 轴 -1~1
Input.get_vector("l","r","u","d")   # 2D 方向向量

Input.is_key_pressed(KEY_SPACE)
Input.get_mouse_button_mask()
Input.get_joy_axis(0, JOY_AXIS_LEFT_X)
Input.is_joy_button_pressed(0, JOY_BUTTON_A)

# ── 事件类型判断 ───────────────────────────────────
event is InputEventKey
event is InputEventMouseButton
event is InputEventMouseMotion
event is InputEventJoypadButton
event is InputEventJoypadMotion
event is InputEventScreenTouch
event is InputEventScreenDrag

# ── 鼠标 ──────────────────────────────────────────
Input.mouse_mode = Input.MOUSE_MODE_CAPTURED
get_viewport().get_mouse_position()
get_global_mouse_position()         # 在 Node2D 中

# ── 手柄 ──────────────────────────────────────────
Input.get_connected_joypads()
Input.get_joy_name(device_id)
Input.start_joy_vibration(0, 0.5, 0.5, 0.3)

# ── InputMap ──────────────────────────────────────
InputMap.has_action("jump")
InputMap.add_action("x", deadzone)
InputMap.action_add_event("x", event)
InputMap.action_erase_events("x")
InputMap.action_get_events("x")

# ── 其他 ──────────────────────────────────────────
get_viewport().set_input_as_handled()  # 阻止事件继续传播
event.as_text()                         # 事件的可读字符串
```
