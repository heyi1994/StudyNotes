# GDScript


---

## 1. 简介

GDScript 是 Godot 引擎的专属脚本语言，语法类似 Python，专为游戏开发设计。

**主要特点：**

- 动态类型（支持可选静态类型注解）
- 与 Godot 节点系统深度集成
- 支持协程（yield/await）
- 热重载友好
- 无需编译，开发迭代快

**文件扩展名：** `.gd`

**基本脚本结构：**

```gdscript
extends Node          # 继承的节点类型

class_name MyClass    # 可选：注册为全局类名

# 成员变量
var health: int = 100

# 生命周期函数
func _ready() -> void:
    print("节点就绪")

func _process(delta: float) -> void:
    pass
```

---

## 2. 基础语法

### 2.1 缩进

GDScript 使用 **Tab 缩进**（不使用花括号），类似 Python：

```gdscript
func example():
    if true:
        print("缩进正确")
```

### 2.2 注释

```gdscript
# 这是单行注释

## 这是文档注释（用于工具提示）

# GDScript 没有原生多行注释，用多个 # 代替
# 第一行
# 第二行
```

### 2.3 分号与换行

```gdscript
var a = 1   # 行尾不需要分号
var b = 2; var c = 3  # 同一行多条语句用分号分隔（不推荐）

# 长行可用 \ 续行
var result = 1 + 2 + \
             3 + 4
```

### 2.4 保留关键字

```
if elif else for while match break continue
return pass and or not in is as
var const enum func class extends signal
true false null self super
await yield
```

---

## 3. 变量与数据类型

### 3.1 变量声明

```gdscript
# 动态类型
var name = "Alice"
var score = 0
var is_alive = true

# 静态类型（推荐，提高性能和可读性）
var health: int = 100
var speed: float = 3.5
var player_name: String = "Bob"
var active: bool = false

# 类型推断（使用 := ）
var level := 1          # 推断为 int
var ratio := 0.5        # 推断为 float
var tag := "hero"       # 推断为 String
```

### 3.2 常量

```gdscript
const MAX_HEALTH = 200
const GRAVITY: float = 9.8
const PLAYER_NAME: String = "Hero"
```

### 3.3 基本数据类型

| 类型         | 说明       | 示例                |
| ------------ | ---------- | ------------------- |
| `int`        | 整数       | `42`, `-10`         |
| `float`      | 浮点数     | `3.14`, `-0.5`      |
| `bool`       | 布尔       | `true`, `false`     |
| `String`     | 字符串     | `"hello"`           |
| `Vector2`    | 二维向量   | `Vector2(1.0, 2.0)` |
| `Vector3`    | 三维向量   | `Vector3(1, 2, 3)`  |
| `Color`      | 颜色       | `Color(1, 0, 0, 1)` |
| `Array`      | 数组       | `[1, 2, 3]`         |
| `Dictionary` | 字典       | `{"key": "value"}`  |
| `Callable`   | 可调用对象 | `func_name`         |
| `null`       | 空值       | `null`              |

### 3.4 类型转换

```gdscript
var i: int = 10
var f: float = float(i)     # int -> float
var s: String = str(i)      # int -> String
var i2: int = int(f)        # float -> int（截断）
var f2: float = "3.14".to_float()  # String -> float
var i3: int = "42".to_int()        # String -> int

# 使用 as 进行安全类型转换（失败返回 null）
var node = get_node("Player") as CharacterBody2D
```

### 3.5 枚举

```gdscript
# 匿名枚举
enum { IDLE, WALK, RUN, JUMP }

# 命名枚举
enum State { IDLE, WALK, RUN, JUMP }

# 带自定义值
enum Direction {
    NORTH = 0,
    SOUTH = 1,
    EAST  = 2,
    WEST  = 3,
}

var current_state: State = State.IDLE
print(State.RUN)          # 2
print(State.keys())       # ["IDLE", "WALK", "RUN", "JUMP"]
```

### 3.6 变量作用域

```gdscript
var global_var = "全局"   # 类成员变量

func my_func():
    var local_var = "局部"  # 函数内局部变量
    print(global_var)       # 可以访问成员变量
    print(local_var)        # 可以访问局部变量

# for 循环变量也是局部作用域
for i in range(5):
    var loop_var = i
# 此处 i 和 loop_var 均不可访问
```

---

## 4. 运算符

### 4.1 算术运算符

```gdscript
var a = 10
var b = 3

print(a + b)    # 13  加
print(a - b)    # 7   减
print(a * b)    # 30  乘
print(a / b)    # 3   整数除法（两个int相除）
print(10.0 / 3) # 3.333...  浮点除法
print(a % b)    # 1   取余
print(a ** b)   # 1000  幂运算（Godot 4）
```

### 4.2 比较运算符

```gdscript
print(5 == 5)   # true
print(5 != 3)   # true
print(5 > 3)    # true
print(5 < 3)    # false
print(5 >= 5)   # true
print(5 <= 4)   # false
```

### 4.3 逻辑运算符

```gdscript
print(true and false)   # false
print(true or false)    # true
print(not true)         # false

# 也可以使用符号形式
print(true && false)    # false
print(true || false)    # true
print(!true)            # false
```

### 4.4 位运算符

```gdscript
print(5 & 3)    # 1   按位与
print(5 | 3)    # 7   按位或
print(5 ^ 3)    # 6   按位异或
print(~5)       # -6  按位取反
print(5 << 1)   # 10  左移
print(5 >> 1)   # 2   右移
```

### 4.5 赋值运算符

```gdscript
var x = 10
x += 5   # x = 15
x -= 3   # x = 12
x *= 2   # x = 24
x /= 4   # x = 6
x %= 4   # x = 2
x **= 3  # x = 8
```

### 4.6 特殊运算符

```gdscript
# in 运算符
print(3 in [1, 2, 3])       # true
print("a" in {"a": 1})      # true
print("hello" in "hello world")  # true

# is 运算符（类型检查）
var node = Node.new()
print(node is Node)          # true
print(node is Object)        # true

# as 运算符（类型转换）
var sprite = get_node("Sprite") as Sprite2D
```

---

## 5. 控制流

### 5.1 if / elif / else

```gdscript
var score = 85

if score >= 90:
    print("优秀")
elif score >= 80:
    print("良好")
elif score >= 60:
    print("及格")
else:
    print("不及格")

# 单行 if
if score > 0: print("正分")

# 三元表达式
var result = "通过" if score >= 60 else "失败"
```

### 5.2 for 循环

```gdscript
# range 循环
for i in range(5):          # 0, 1, 2, 3, 4
    print(i)

for i in range(2, 8):       # 2, 3, 4, 5, 6, 7
    print(i)

for i in range(0, 10, 2):   # 0, 2, 4, 6, 8
    print(i)

for i in range(10, 0, -1):  # 10, 9, 8, ..., 1
    print(i)

# 遍历数组
var fruits = ["苹果", "香蕉", "橙子"]
for fruit in fruits:
    print(fruit)

# 遍历字典
var data = {"name": "Alice", "age": 25}
for key in data:
    print(key, ": ", data[key])

for key in data.keys():
    print(key)

for value in data.values():
    print(value)

# 带索引遍历（使用 enumerate 技巧）
for i in fruits.size():
    print(i, ": ", fruits[i])
```

### 5.3 while 循环

```gdscript
var count = 0
while count < 5:
    print(count)
    count += 1

# 无限循环
while true:
    if some_condition:
        break
```

### 5.4 break 与 continue

```gdscript
for i in range(10):
    if i == 3:
        continue    # 跳过当前迭代
    if i == 7:
        break       # 退出循环
    print(i)        # 输出：0 1 2 4 5 6
```

### 5.5 match（模式匹配）

GDScript 的 `match` 比 switch 更强大：

```gdscript
var status = "active"

match status:
    "active":
        print("激活状态")
    "inactive", "disabled":    # 多值匹配
        print("非激活状态")
    _:                         # 默认情况（相当于 default）
        print("未知状态")

# 匹配数字
var code = 404
match code:
    200:
        print("成功")
    404:
        print("未找到")
    500:
        print("服务器错误")
    _:
        print("其他错误")

# 匹配数组模式
var point = [1, 2]
match point:
    [0, 0]:
        print("原点")
    [var x, 0]:
        print("X轴上，x =", x)
    [0, var y]:
        print("Y轴上，y =", y)
    [var x, var y]:
        print("坐标 (", x, ",", y, ")")

# 匹配字典
var event = {"type": "click", "button": 1}
match event:
    {"type": "click", "button": var btn}:
        print("点击按钮:", btn)
    {"type": "key", "key": var k}:
        print("按键:", k)
```

---

## 6. 函数

### 6.1 基本函数定义

```gdscript
# 无参数无返回值
func greet() -> void:
    print("你好！")

# 带参数
func add(a: int, b: int) -> int:
    return a + b

# 带默认参数
func greet_player(name: String = "玩家", level: int = 1) -> String:
    return "欢迎，" + name + "！等级：" + str(level)

# 调用
greet()
var result = add(3, 4)       # result = 7
var msg = greet_player()     # 使用默认参数
var msg2 = greet_player("Alice", 5)
```

### 6.2 可变参数

```gdscript
func sum_all(...) -> int:
    var total = 0
    for n in arguments:
        total += n
    return total

# Godot 4 推荐使用数组参数代替变参
func sum_array(numbers: Array) -> int:
    var total = 0
    for n in numbers:
        total += n
    return total
```

### 6.3 返回多个值

```gdscript
# 通过数组或字典返回多个值
func get_position() -> Array:
    return [100.0, 200.0]

func get_player_info() -> Dictionary:
    return {"name": "Alice", "health": 100, "level": 5}

var pos = get_position()
print(pos[0], pos[1])

var info = get_player_info()
print(info["name"])
```

### 6.4 Lambda 函数（Callable）

```gdscript
# 使用 func 关键字创建 lambda
var multiply = func(a: int, b: int) -> int:
    return a * b

print(multiply.call(3, 4))   # 12

# 作为参数传递
var numbers = [3, 1, 4, 1, 5, 9]
numbers.sort_custom(func(a, b): return a > b)  # 降序排列
print(numbers)   # [9, 5, 4, 3, 1, 1]
```

### 6.5 静态函数

```gdscript
class_name MathUtils

static func clamp_value(value: float, min_val: float, max_val: float) -> float:
    return clamp(value, min_val, max_val)

# 调用（不需要实例）
var v = MathUtils.clamp_value(150.0, 0.0, 100.0)  # 100.0
```

### 6.6 虚函数（可重写函数）

```gdscript
# 父类
class Animal:
    func speak() -> String:
        return "..."

# 子类重写
class Dog extends Animal:
    func speak() -> String:
        return "汪汪！"

class Cat extends Animal:
    func speak() -> String:
        return "喵喵！"
```

---

## 7. 类与面向对象

### 7.1 基本类结构

```gdscript
extends Node

class_name Player

# 成员变量
var health: int = 100
var max_health: int = 100
var speed: float = 200.0
var player_name: String = ""

# 静态变量（所有实例共享）
static var player_count: int = 0

# 构造函数
func _init(name: String = "Player") -> void:
    player_name = name
    player_count += 1
    print("创建玩家：", name)

# 析构（对象被释放时）
func _notification(what: int) -> void:
    if what == NOTIFICATION_PREDELETE:
        player_count -= 1
        print("销毁玩家：", player_name)

# 方法
func heal(amount: int) -> void:
    health = min(health + amount, max_health)

func take_damage(amount: int) -> void:
    health = max(health - amount, 0)
    if health == 0:
        die()

func is_alive() -> bool:
    return health > 0

func die() -> void:
    print(player_name, " 已死亡")
    queue_free()
```

### 7.2 继承

```gdscript
extends Node

class_name Animal

var name: String = ""
var sound: String = "..."

func _init(n: String) -> void:
    name = n

func speak() -> void:
    print(name, " 说：", sound)

func describe() -> String:
    return "动物：" + name
```

```gdscript
extends Animal

class_name Dog

func _init(n: String) -> void:
    super._init(n)    # 调用父类构造函数
    sound = "汪汪！"

# 重写父类方法
func speak() -> void:
    super.speak()     # 可选：调用父类实现
    print("（狗在摇尾巴）")

func fetch() -> void:
    print(name, " 去捡球了！")
```

### 7.3 内部类

```gdscript
extends Node

class Item:
    var item_name: String
    var value: int

    func _init(n: String, v: int) -> void:
        item_name = n
        value = v

    func describe() -> String:
        return item_name + " (价值: " + str(value) + ")"

# 使用内部类
func _ready() -> void:
    var sword = Item.new("铁剑", 100)
    print(sword.describe())
```

### 7.4 属性（get/set）

```gdscript
extends Node

var _health: int = 100

# Godot 4 属性语法
var health: int:
    get:
        return _health
    set(value):
        _health = clamp(value, 0, 100)
        health_changed.emit(_health)    # 触发信号

signal health_changed(new_health: int)

# 只读属性
var is_alive: bool:
    get:
        return _health > 0
```

### 7.5 接口模拟（鸭子类型）

GDScript 没有接口，但可以用 `has_method` 模拟：

```gdscript
func interact(obj: Node) -> void:
    if obj.has_method("on_interact"):
        obj.on_interact(self)
    else:
        push_warning("对象不支持交互：" + obj.name)
```

---

## 8. 信号（Signal）

信号是 Godot 的观察者模式实现，用于节点间通信。

### 8.1 定义和发射信号

```gdscript
extends Node

# 定义信号
signal health_changed(old_value: int, new_value: int)
signal player_died
signal item_collected(item_name: String, item_value: int)

var health: int = 100

func take_damage(amount: int) -> void:
    var old_health = health
    health -= amount
    health_changed.emit(old_health, health)  # 发射信号

    if health <= 0:
        player_died.emit()
```

### 8.2 连接信号

```gdscript
# 方式一：在代码中连接
func _ready() -> void:
    var player = get_node("Player")
    player.health_changed.connect(_on_player_health_changed)
    player.player_died.connect(_on_player_died)

func _on_player_health_changed(old_val: int, new_val: int) -> void:
    print("血量从 ", old_val, " 变为 ", new_val)

func _on_player_died() -> void:
    print("玩家死亡！")

# 方式二：连接一次性信号（触发后自动断开）
player.player_died.connect(_on_player_died, CONNECT_ONE_SHOT)

# 方式三：延迟信号（下一帧触发）
player.health_changed.connect(_on_health_changed, CONNECT_DEFERRED)
```

### 8.3 断开信号

```gdscript
func cleanup() -> void:
    var player = get_node("Player")
    if player.health_changed.is_connected(_on_player_health_changed):
        player.health_changed.disconnect(_on_player_health_changed)
```

### 8.4 await 等待信号

```gdscript
func wait_for_player_death() -> void:
    print("等待玩家死亡...")
    await player.player_died
    print("玩家已死亡，继续执行")

# 等待计时器
func timed_action() -> void:
    await get_tree().create_timer(2.0).timeout
    print("2秒后执行")
```

### 8.5 内置常用信号

```gdscript
# 节点信号
Node.ready             # 节点就绪
Node.tree_entered      # 进入场景树
Node.tree_exited       # 退出场景树

# 场景树信号
SceneTree.process_frame     # 每帧（process后）
SceneTree.physics_frame     # 每物理帧

# Timer 信号
Timer.timeout              # 计时器超时

# 按钮信号
Button.pressed             # 按钮被按下
Button.toggled(on: bool)   # 切换按钮状态改变

# 输入信号（LineEdit）
LineEdit.text_changed(new_text: String)
LineEdit.text_submitted(new_text: String)
```

---

## 9. 数组与字典

### 9.1 数组（Array）

```gdscript
# 创建数组
var arr = [1, 2, 3, 4, 5]
var empty: Array = []
var typed: Array[int] = [1, 2, 3]    # Godot 4 类型化数组

# 访问元素
print(arr[0])       # 1
print(arr[-1])      # 5（负索引从尾部访问）
print(arr.front())  # 1（第一个）
print(arr.back())   # 5（最后一个）

# 修改元素
arr[0] = 10

# 添加元素
arr.append(6)           # 末尾添加
arr.push_back(7)        # 等同于 append
arr.push_front(0)       # 头部添加
arr.insert(2, 99)       # 在索引2处插入

# 删除元素
arr.pop_back()          # 删除并返回最后一个
arr.pop_front()         # 删除并返回第一个
arr.remove_at(0)        # 删除索引0的元素
arr.erase(99)           # 删除值为99的第一个元素

# 查找
print(arr.has(3))           # 是否包含3
print(arr.find(3))          # 返回索引，没有返回 -1
print(arr.count(2))         # 值2出现的次数

# 排序
arr.sort()                  # 升序排序
arr.sort_custom(func(a,b): return a > b)  # 自定义排序

# 数组操作
var arr2 = [4, 5, 6]
var combined = arr + arr2   # 合并数组
arr.append_array(arr2)      # 追加数组

# 切片
var sliced = arr.slice(1, 3)    # 获取索引1到2的子数组

# 其他操作
print(arr.size())           # 数组长度
arr.reverse()               # 反转数组
arr.shuffle()               # 随机打乱
arr.clear()                 # 清空数组
var copy = arr.duplicate()  # 浅拷贝
var deep_copy = arr.duplicate(true)  # 深拷贝

# 过滤与映射（Godot 4）
var evens = arr.filter(func(n): return n % 2 == 0)
var doubled = arr.map(func(n): return n * 2)
var total = arr.reduce(func(acc, n): return acc + n, 0)
```

### 9.2 字典（Dictionary）

```gdscript
# 创建字典
var dict = {"name": "Alice", "age": 25, "score": 100}
var empty_dict: Dictionary = {}

# 访问
print(dict["name"])           # Alice
print(dict.name)              # Alice（点语法，键为合法标识符时可用）
print(dict.get("name", ""))   # 安全访问，不存在时返回默认值

# 修改/添加
dict["name"] = "Bob"
dict["level"] = 5
dict.new_key = "new_value"

# 删除
dict.erase("age")

# 检查
print(dict.has("name"))       # true
print(dict.has_all(["name", "score"]))  # 检查多个键

# 遍历
for key in dict:
    print(key, ": ", dict[key])

for key in dict.keys():
    print(key)

for value in dict.values():
    print(value)

# 其他操作
print(dict.size())           # 键值对数量
dict.clear()                 # 清空
var copy = dict.duplicate()  # 拷贝
var keys = dict.keys()       # 获取所有键（Array）
var vals = dict.values()     # 获取所有值（Array）
dict.merge({"x": 1, "y": 2})          # 合并（不覆盖已有键）
dict.merge({"name": "Charlie"}, true)  # 合并并覆盖
```

---

## 10. 字符串操作

```gdscript
var s = "Hello, World!"

# 基本操作
print(s.length())           # 13
print(s.to_upper())         # HELLO, WORLD!
print(s.to_lower())         # hello, world!

# 查找
print(s.find("World"))      # 7（返回索引）
print(s.contains("Hello"))  # true
print(s.begins_with("He"))  # true
print(s.ends_with("!"))     # true

# 截取
print(s.substr(7, 5))       # World
print(s.left(5))            # Hello
print(s.right(6))           # orld!

# 替换
print(s.replace("World", "GDScript"))  # Hello, GDScript!

# 分割与合并
var parts = "a,b,c".split(",")         # ["a", "b", "c"]
var joined = ",".join(["a", "b", "c"]) # "a,b,c"

# 去除空白
var padded = "  hello  "
print(padded.strip_edges())            # "hello"
print(padded.lstrip())                 # "hello  "
print(padded.rstrip())                 # "  hello"

# 格式化
var name = "Alice"
var level = 5
# 字符串插值（Godot 4）
print("玩家 %s 等级 %d" % [name, level])

# 类型转换
print("42".to_int())         # 42
print("3.14".to_float())     # 3.14
print(str(42))               # "42"

# 字符操作
print(s[0])                  # H
print(s.unicode_at(0))       # 72（H 的 Unicode）
print(char(72))              # H

# 检查
print("abc123".is_valid_identifier())   # true
print("123".is_valid_int())             # true
print("3.14".is_valid_float())          # true
```

---

## 11. 节点与场景

### 11.1 节点访问

```gdscript
extends Node

func _ready() -> void:
    # 通过路径访问
    var child = get_node("ChildNode")
    var nested = get_node("Parent/Child/GrandChild")

    # 简写语法（$）
    var label = $Label
    var button = $UI/Button

    # 带类型断言
    var sprite = $Sprite2D as Sprite2D

    # 查找子节点
    var found = find_child("PlayerSprite")         # 按名称递归查找
    var found_type = find_children("*", "Sprite2D")  # 按类型查找

    # 访问父节点
    var parent = get_parent()

    # 访问根节点
    var root = get_tree().root

    # 检查节点存在
    if has_node("OptionalNode"):
        var optional = get_node("OptionalNode")
```

### 11.2 节点生命周期

```gdscript
extends Node

# 节点被添加到场景树后调用（初始化）
func _ready() -> void:
    print("_ready: ", name)

# 每帧调用（处理游戏逻辑）
func _process(delta: float) -> void:
    pass  # delta = 距上一帧的时间（秒）

# 每物理帧调用（固定时间步，默认60次/秒）
func _physics_process(delta: float) -> void:
    pass

# 处理输入事件
func _input(event: InputEvent) -> void:
    if event is InputEventKey:
        pass

# 未处理的输入（GUI控件未消耗的）
func _unhandled_input(event: InputEvent) -> void:
    pass

# 通知回调
func _notification(what: int) -> void:
    match what:
        NOTIFICATION_READY:
            pass
        NOTIFICATION_PROCESS:
            pass
        NOTIFICATION_ENTER_TREE:
            pass
        NOTIFICATION_EXIT_TREE:
            pass
```

### 11.3 动态创建和销毁节点

```gdscript
extends Node

# 动态创建节点
func spawn_enemy() -> void:
    var enemy_scene = preload("res://scenes/Enemy.tscn")
    var enemy = enemy_scene.instantiate()
    enemy.position = Vector2(100, 200)
    enemy.name = "Enemy_" + str(randi())
    add_child(enemy)

# 销毁节点
func destroy_node(node: Node) -> void:
    node.queue_free()    # 安全删除（本帧末尾执行）
    # node.free()        # 立即删除（小心使用）

# 移动节点
func reparent_node(node: Node, new_parent: Node) -> void:
    node.reparent(new_parent)
```

### 11.4 场景切换

```gdscript
extends Node

# 切换主场景
func go_to_main_menu() -> void:
    get_tree().change_scene_to_file("res://scenes/MainMenu.tscn")

# 使用打包场景切换
func go_to_game() -> void:
    var scene = preload("res://scenes/Game.tscn")
    get_tree().change_scene_to_packed(scene)

# 重载当前场景
func restart() -> void:
    get_tree().reload_current_scene()

# 退出游戏
func quit() -> void:
    get_tree().quit()
```

### 11.5 Groups（组）

```gdscript
extends Node

func _ready() -> void:
    add_to_group("enemies")
    add_to_group("damageable")

# 通过组操作所有节点
func damage_all_enemies(damage: int) -> void:
    get_tree().call_group("enemies", "take_damage", damage)

# 获取组内所有节点
func get_all_enemies() -> Array:
    return get_tree().get_nodes_in_group("enemies")

# 检查是否在某个组
func check_group() -> void:
    if is_in_group("enemies"):
        print("是敌人")
```

---

## 12. 输入处理

### 12.1 轮询输入（推荐用于移动控制）

```gdscript
extends CharacterBody2D

const SPEED = 200.0

func _physics_process(delta: float) -> void:
    var direction = Vector2.ZERO

    # 检查输入动作（在 Project Settings -> Input Map 配置）
    if Input.is_action_pressed("ui_right"):
        direction.x += 1
    if Input.is_action_pressed("ui_left"):
        direction.x -= 1
    if Input.is_action_pressed("ui_down"):
        direction.y += 1
    if Input.is_action_pressed("ui_up"):
        direction.y -= 1

    # 更简洁的写法
    direction = Input.get_vector("ui_left", "ui_right", "ui_up", "ui_down")

    velocity = direction.normalized() * SPEED
    move_and_slide()

    # 只在按下那一帧触发（单次检测）
    if Input.is_action_just_pressed("jump"):
        jump()

    # 只在松开那一帧触发
    if Input.is_action_just_released("attack"):
        end_attack()
```

### 12.2 事件输入（推荐用于一次性动作）

```gdscript
extends Node

func _input(event: InputEvent) -> void:
    # 键盘事件
    if event is InputEventKey:
        var key_event = event as InputEventKey
        if key_event.pressed and key_event.keycode == KEY_ESCAPE:
            get_tree().quit()

    # 鼠标按钮
    if event is InputEventMouseButton:
        var mouse_event = event as InputEventMouseButton
        if mouse_event.pressed:
            match mouse_event.button_index:
                MOUSE_BUTTON_LEFT:
                    print("左键点击：", mouse_event.position)
                MOUSE_BUTTON_RIGHT:
                    print("右键点击：", mouse_event.position)

    # 鼠标移动
    if event is InputEventMouseMotion:
        var motion = event as InputEventMouseMotion
        print("鼠标位置：", motion.position)
        print("鼠标速度：", motion.velocity)

    # 动作事件（推荐方式）
    if event.is_action_pressed("attack"):
        attack()
```

### 12.3 鼠标位置

```gdscript
# 全局鼠标位置
var global_mouse = get_viewport().get_mouse_position()

# 相对于当前节点的本地鼠标位置
var local_mouse = get_local_mouse_position()
```

---

## 13. 协程与异步

### 13.1 await 基础

```gdscript
extends Node

func _ready() -> void:
    print("开始")
    await do_async_task()
    print("任务完成")

func do_async_task() -> void:
    print("等待 2 秒...")
    await get_tree().create_timer(2.0).timeout
    print("2 秒到了")
```

### 13.2 链式 await

```gdscript
func sequence_of_events() -> void:
    # 按顺序执行
    await get_tree().create_timer(1.0).timeout
    print("1秒后")

    await get_tree().create_timer(1.0).timeout
    print("2秒后")

    await get_tree().create_timer(1.0).timeout
    print("3秒后")
```

### 13.3 等待信号

```gdscript
func wait_for_button() -> void:
    print("等待按钮点击...")
    await $Button.pressed
    print("按钮被按下！")

# 等待动画完成
func play_and_wait() -> void:
    $AnimationPlayer.play("attack")
    await $AnimationPlayer.animation_finished
    print("攻击动画结束")
```

### 13.4 并行异步（自定义信号协调）

```gdscript
var _tasks_done = 0
signal all_done

func run_parallel() -> void:
    _tasks_done = 0
    task_a()
    task_b()
    await all_done
    print("所有任务完成")

func task_a() -> void:
    await get_tree().create_timer(1.0).timeout
    print("任务A完成")
    _tasks_done += 1
    if _tasks_done == 2:
        all_done.emit()

func task_b() -> void:
    await get_tree().create_timer(2.0).timeout
    print("任务B完成")
    _tasks_done += 1
    if _tasks_done == 2:
        all_done.emit()
```

---

## 14. 资源与文件操作

### 14.1 加载资源

```gdscript
# 预加载（编译时加载，适合小资源）
var texture = preload("res://assets/player.png")
var scene = preload("res://scenes/Enemy.tscn")

# 动态加载（运行时加载）
var texture2 = load("res://assets/player.png")

# 异步加载（Godot 4）
func load_async() -> void:
    ResourceLoader.load_threaded_request("res://assets/big_texture.png")
    await get_tree().process_frame
    var status = ResourceLoader.load_threaded_get_status("res://assets/big_texture.png")
    if status == ResourceLoader.THREAD_LOAD_LOADED:
        var res = ResourceLoader.load_threaded_get("res://assets/big_texture.png")
```

### 14.2 文件读写

```gdscript
# 写入文件
func save_data(data: Dictionary) -> void:
    var file = FileAccess.open("user://save_data.json", FileAccess.WRITE)
    if file:
        file.store_string(JSON.stringify(data, "\t"))
        file.close()
    else:
        push_error("无法打开文件写入")

# 读取文件
func load_data() -> Dictionary:
    if not FileAccess.file_exists("user://save_data.json"):
        return {}

    var file = FileAccess.open("user://save_data.json", FileAccess.READ)
    if file:
        var content = file.get_as_text()
        file.close()
        var json = JSON.new()
        var err = json.parse(content)
        if err == OK:
            return json.data
    return {}

# 路径说明
# res:// = 项目资源目录（只读，打包后无法写入）
# user:// = 用户数据目录（可读写，跨平台安全路径）
```

### 14.3 自定义资源

```gdscript
# 定义自定义资源类
extends Resource

class_name ItemData

@export var item_name: String = ""
@export var item_icon: Texture2D
@export var value: int = 0
@export var description: String = ""

func get_display_name() -> String:
    return "[%s] %s" % [value, item_name]
```

---

## 15. 常用内置类型

### 15.1 Vector2 / Vector3

```gdscript
var v2 = Vector2(3.0, 4.0)

print(v2.x, v2.y)              # 3.0 4.0
print(v2.length())             # 5.0
print(v2.normalized())         # Vector2(0.6, 0.8)
print(v2.distance_to(Vector2(0, 0)))  # 5.0
print(v2.dot(Vector2(1, 0)))   # 3.0（点积）
print(v2.angle())              # 角度（弧度）
print(v2.rotated(PI / 2))      # 旋转90度

# 常用常量
Vector2.ZERO    # (0, 0)
Vector2.ONE     # (1, 1)
Vector2.UP      # (0, -1)
Vector2.DOWN    # (0, 1)
Vector2.LEFT    # (-1, 0)
Vector2.RIGHT   # (1, 0)

# 线性插值
var start = Vector2(0, 0)
var end = Vector2(100, 100)
var mid = start.lerp(end, 0.5)  # Vector2(50, 50)

# Vector3
var v3 = Vector3(1, 2, 3)
print(v3.cross(Vector3(4, 5, 6)))  # 叉积
```

### 15.2 Color

```gdscript
# 创建颜色
var red = Color(1, 0, 0)           # RGB
var semi_transparent = Color(1, 0, 0, 0.5)  # RGBA
var from_hex = Color.hex(0xFF5733FF)
var from_name = Color.BLUE          # 预定义颜色
var from_html = Color.from_string("#FF5733", Color.WHITE)

# 属性
print(red.r, red.g, red.b, red.a)  # 1 0 0 1
print(red.h, red.s, red.v)          # HSV 值

# 操作
var lighter = red.lightened(0.3)
var darker = red.darkened(0.3)
var mixed = red.lerp(Color.BLUE, 0.5)  # 混合

# 转换
print(red.to_html())         # "ff0000ff"
print(red.to_html(false))    # "ff0000"（不含alpha）
```

### 15.3 Rect2

```gdscript
var rect = Rect2(Vector2(10, 20), Vector2(100, 50))
# 或
var rect2 = Rect2(10, 20, 100, 50)  # x, y, width, height

print(rect.position)        # Vector2(10, 20)
print(rect.size)            # Vector2(100, 50)
print(rect.end)             # Vector2(110, 70)（右下角）

# 检测
print(rect.has_point(Vector2(50, 40)))   # true（点在矩形内）
print(rect.intersects(Rect2(50, 50, 100, 100)))  # true（矩形相交）

# 操作
var expanded = rect.expand(Vector2(200, 200))  # 扩展到包含该点
var grown = rect.grow(10)                      # 四周扩大10
var merged = rect.merge(Rect2(200, 200, 50, 50))  # 合并为包含两者的最小矩形
```

### 15.4 Transform2D / Transform3D

```gdscript
# 2D 变换
var t = Transform2D.IDENTITY

# 创建变换
t = Transform2D(rotation, position)

# 操作
t = t.rotated(PI / 4)           # 旋转45度
t = t.scaled(Vector2(2, 2))     # 缩放
t = t.translated(Vector2(100, 0))  # 平移

# 应用变换
var point = Vector2(10, 0)
var transformed = t * point

# 节点变换
$Node.global_transform
$Node.global_position
$Node.global_rotation
$Node.global_scale
```

---

## 16. 调试与工具

### 16.1 打印输出

```gdscript
print("普通输出")
print("多个值：", 1, " ", true, " ", Vector2.ONE)
print_rich("[color=red]红色文本[/color]")  # 富文本输出

printerr("错误输出（红色）")
push_error("记录错误（不中断）")
push_warning("记录警告")

# 格式化输出
print("格式化：%s = %d" % ["分数", 100])
print("浮点：%.2f" % 3.14159)  # 3.14
```

### 16.2 断言

```gdscript
assert(health >= 0, "血量不能为负数！")
assert(speed > 0)  # 简单断言（调试模式才有效）
```

### 16.3 @tool 注解

```gdscript
@tool  # 使脚本在编辑器中运行
extends Node2D

@export var my_value: int = 0:
    set(v):
        my_value = v
        update_configuration_warnings()

# 编辑器中显示警告
func _get_configuration_warnings() -> PackedStringArray:
    var warnings = PackedStringArray()
    if my_value <= 0:
        warnings.append("my_value 必须大于0")
    return warnings
```

### 16.4 常用注解

```gdscript
@export var speed: float = 200.0           # 在检查器中暴露
@export_range(0, 100) var health: int = 100  # 范围限制
@export_enum("Easy", "Normal", "Hard") var difficulty: int = 1
@export_group("战斗属性")                 # 分组
@export var damage: int = 10
@export var defense: int = 5
@export_subgroup("高级")
@export var crit_chance: float = 0.1

@onready var sprite = $Sprite2D           # ready时自动赋值
@static_unload                            # 静态变量不持久化
@warning_ignore("unused_variable")        # 忽略警告

# 函数注解
@rpc                                      # 远程过程调用（多人游戏）
@rpc("authority")
func sync_position(pos: Vector2) -> void:
    position = pos
```

---

## 17. 最佳实践

### 17.1 命名规范

```gdscript
# 变量和函数：snake_case
var player_health: int = 100
func calculate_damage() -> int:
    return 0

# 常量和枚举值：SCREAMING_SNAKE_CASE
const MAX_SPEED = 500
enum State { IDLE, RUNNING, JUMPING }

# 类名：PascalCase
class_name PlayerCharacter

# 信号：snake_case（通常以动词或事件描述）
signal health_changed(new_health: int)
signal enemy_died(enemy: Node)

# 私有成员（约定，无强制）：_下划线前缀
var _internal_timer: float = 0.0
func _update_internal_state() -> void:
    pass
```

### 17.2 性能优化

```gdscript
extends Node

# 避免在 _process 中频繁使用 get_node()，改用 @onready
@onready var label: Label = $Label
@onready var player: CharacterBody2D = $Player

# 缓存频繁使用的值
var _cached_transform: Transform2D

func _ready() -> void:
    _cached_transform = global_transform

# 使用 is_instance_valid() 检查节点有效性（而非 != null）
func safe_access(node: Node) -> void:
    if is_instance_valid(node):
        node.do_something()

# 预加载场景（而非每次 load）
const BULLET_SCENE = preload("res://scenes/Bullet.tscn")

# 对象池模式（避免频繁实例化/销毁）
var _bullet_pool: Array[Node] = []

func get_bullet() -> Node:
    for bullet in _bullet_pool:
        if not bullet.visible:
            bullet.visible = true
            return bullet
    var new_bullet = BULLET_SCENE.instantiate()
    add_child(new_bullet)
    _bullet_pool.append(new_bullet)
    return new_bullet
```

### 17.3 代码组织

```gdscript
extends CharacterBody2D

class_name Player

# ================================
# 信号
# ================================
signal health_changed(new_health: int)
signal died

# ================================
# 常量
# ================================
const MAX_HEALTH: int = 100
const SPEED: float = 200.0

# ================================
# 导出变量（Inspector 可见）
# ================================
@export var jump_force: float = 400.0
@export var gravity: float = 980.0

# ================================
# 公共变量
# ================================
var health: int = MAX_HEALTH

# ================================
# 私有变量
# ================================
@onready var _sprite: Sprite2D = $Sprite2D
@onready var _anim: AnimationPlayer = $AnimationPlayer

var _is_dead: bool = false

# ================================
# 生命周期函数
# ================================
func _ready() -> void:
    _setup()

func _process(delta: float) -> void:
    _update_animation()

func _physics_process(delta: float) -> void:
    _handle_movement(delta)
    _apply_gravity(delta)
    move_and_slide()

# ================================
# 公共方法
# ================================
func take_damage(amount: int) -> void:
    if _is_dead:
        return
    health = max(0, health - amount)
    health_changed.emit(health)
    if health == 0:
        _die()

# ================================
# 私有方法
# ================================
func _setup() -> void:
    health = MAX_HEALTH

func _handle_movement(delta: float) -> void:
    var dir = Input.get_axis("ui_left", "ui_right")
    velocity.x = dir * SPEED

func _apply_gravity(delta: float) -> void:
    if not is_on_floor():
        velocity.y += gravity * delta

func _update_animation() -> void:
    if velocity.x != 0:
        _anim.play("walk")
    else:
        _anim.play("idle")

func _die() -> void:
    _is_dead = true
    died.emit()
    queue_free()
```

### 17.4 常见坑与注意事项

```gdscript
# 1. 整数除法陷阱
var a = 5 / 2       # 结果是 2（整数除法！）
var b = 5.0 / 2     # 结果是 2.5（浮点除法）
var c = 5 / 2.0     # 结果是 2.5

# 2. 字典和数组是引用类型
var arr1 = [1, 2, 3]
var arr2 = arr1      # arr2 是 arr1 的引用！
arr2.append(4)
print(arr1)          # [1, 2, 3, 4]  arr1 也被修改了

var arr3 = arr1.duplicate()  # 真正的拷贝

# 3. 浮点比较
var f = 0.1 + 0.2
print(f == 0.3)      # false（浮点精度问题）
print(is_equal_approx(f, 0.3))  # true（使用近似比较）

# 4. null 检查
var node = get_node_or_null("OptionalNode")
if node != null:     # 安全
    node.do_something()

# 5. queue_free 后不要访问节点
var enemy = $Enemy
enemy.queue_free()
# enemy.position = ...  # 错误！节点已标记为删除

# 6. 在信号回调中修改发射者
# 使用 call_deferred 延迟执行避免回调中修改信号发射者导致的问题
func _on_enemy_died() -> void:
    call_deferred("_cleanup_enemy")

func _cleanup_enemy() -> void:
    # 安全地清理
    pass
```

---

## 附录：常用代码片段

### 跟随鼠标旋转

```gdscript
func _process(_delta: float) -> void:
    look_at(get_global_mouse_position())
```

### 平滑移动

```gdscript
func _process(delta: float) -> void:
    var target = get_global_mouse_position()
    global_position = global_position.lerp(target, 5.0 * delta)
```

### 计时器

```gdscript
var _timer: float = 0.0
const INTERVAL: float = 1.0

func _process(delta: float) -> void:
    _timer += delta
    if _timer >= INTERVAL:
        _timer = 0.0
        _on_timer_tick()

func _on_timer_tick() -> void:
    print("每秒触发一次")
```

### 屏幕震动

```gdscript
func shake(duration: float, intensity: float) -> void:
    var original = position
    var elapsed = 0.0
    while elapsed < duration:
        var delta = await get_tree().process_frame
        elapsed += get_process_delta_time()
        position = original + Vector2(
            randf_range(-intensity, intensity),
            randf_range(-intensity, intensity)
        )
    position = original
```

### 单例（Autoload）模式

```gdscript
# 在 Project Settings -> Autoload 中注册为 "GameManager"
extends Node

var score: int = 0
var current_level: int = 1

func add_score(points: int) -> void:
    score += points

# 在任意脚本中访问
# GameManager.add_score(100)
```

