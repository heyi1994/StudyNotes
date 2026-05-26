# Godot .tres

## 1. 什么是 .tres

`.tres` 是 Godot 引擎专用的**文本格式资源文件**（Text Resource），用于持久化存储游戏数据。

- `.tres` = **T**ext **RES**ource
- 对应的二进制版本是 `.res`（Binary Resource）
- 本质上是 Godot `Resource` 类的序列化表示

### 类比理解

| 你熟悉的概念           | Godot 对应              |
| ---------------------- | ----------------------- |
| JSON 配置文件          | .tres                   |
| Unity ScriptableObject | Custom Resource + .tres |
| 数据库一行记录         | 一个 .tres 文件         |
| Excel 表格模板         | Resource 类定义         |
| Excel 表格一行         | 一个 .tres 实例         |

---

## 2. Resource 基础概念

在 Godot 中，`Resource` 是所有资源的基类。你见过的很多东西本质上都是 Resource：

```
Resource
├── Texture2D        （图片）
├── AudioStream      （音频）
├── Shader           （着色器）
├── Material         （材质）
├── Script           （脚本）
├── PackedScene      （场景，即 .tscn 文件）
└── 你自定义的数据类  （武将、技能、道具...）
```

### Resource 的核心特性

**引用计数共享：** 同一个 .tres 文件在内存中只有一份，多个节点引用同一个资源时，修改会互相影响。

```gdscript
var data_a = load("res://hero.tres")
var data_b = load("res://hero.tres")

# data_a 和 data_b 指向同一个内存对象！
data_a.attack = 999
print(data_b.attack)  # 输出 999
```

**需要独立副本时，使用 `.duplicate()`：**

```gdscript
var data_a = load("res://hero.tres")
var data_b = data_a.duplicate()  # 深拷贝

data_a.attack = 999
print(data_b.attack)  # 输出原始值，不受影响
```

---

## 3. 创建自定义 Resource

### 步骤一：定义 Resource 类

新建一个 GDScript 文件，继承 `Resource`，加上 `class_name`：

```gdscript
# unit_data.gd
class_name UnitData
extends Resource

@export var unit_name: String = ""
@export var max_hp: int = 100
@export var attack: int = 50
@export var defense: int = 30
@export var move_range: int = 4
@export var sprite: Texture2D
```

**关键点：**

- `class_name` 必须写，否则编辑器无法识别这个类型
- `extends Resource` 必须继承 Resource
- `@export` 让字段在编辑器中可见可编辑
- 给字段设默认值是好习惯

### 步骤二：在编辑器中创建实例

1. 在 FileSystem 面板中，右键目标文件夹
2. 选择 **New → Resource**
3. 在弹出框中输入 `UnitData`，选中后点 Create
4. 保存文件，命名为 `关羽.tres`

### 步骤三：填写数据

选中刚创建的 `.tres` 文件，在 Inspector 面板中直接填写各字段值。

---

## 4. .tres 文件结构解析

用文本编辑器打开一个 .tres 文件，内容如下：

```ini
[gd_resource type="UnitData" load_steps=2 format=3 uid="uid://abc123"]

[ext_resource type="Texture2D" path="res://assets/guanyu.png" id="1_xyz"]

[resource]
unit_name = "关羽"
max_hp = 120
attack = 85
defense = 60
move_range = 4
sprite = ExtResource("1_xyz")
```

### 各部分说明

| 部分                            | 说明                                     |
| ------------------------------- | ---------------------------------------- |
| `[gd_resource type="UnitData"]` | 文件头，声明这是什么类型的 Resource      |
| `load_steps=2`                  | 加载时需要几步（包含外部引用时大于1）    |
| `uid="uid://..."`               | Godot 4 的唯一ID，重命名文件后引用不会断 |
| `[ext_resource]`                | 引用的外部文件（图片、音频等）           |
| `[sub_resource]`                | 内嵌的子资源（嵌套 Resource）            |
| `[resource]`                    | 主资源的字段数据                         |

### 支持的数据类型

```ini
[resource]
# 基本类型
my_bool = true
my_int = 42
my_float = 3.14
my_string = "你好"

# Godot 内置类型
my_vector2 = Vector2(100, 200)
my_vector3 = Vector3(1, 2, 3)
my_color = Color(1, 0, 0, 1)       # RGBA，红色
my_rect2 = Rect2(0, 0, 64, 64)

# 数组
my_array = Array[int]([1, 2, 3, 4])
my_string_array = PackedStringArray(["魏", "蜀", "吴"])

# 字典
my_dict = { "key": "value", "number": 42 }

# 外部资源引用
my_texture = ExtResource("1_abc")

# 内嵌子资源
my_skill = SubResource("skill_001")
```

---

## 5. 在编辑器中使用

### Inspector 面板编辑

选中 .tres 文件后，右侧 Inspector 面板会显示所有 `@export` 的字段：

- `int` / `float` → 数字输入框
- `String` → 文本输入框
- `bool` → 勾选框
- `Color` → 颜色选择器
- `Texture2D` → 图片拖放区域
- `Array` → 可展开的列表
- `@export_enum` → 下拉选择框

### 使用 @export 注解增强编辑体验

```gdscript
class_name UnitData
extends Resource

# 下拉枚举
@export_enum("魏", "蜀", "吴", "群") var faction: String = "蜀"

# 整数范围限制（显示为滑块）
@export_range(1, 10) var star_level: int = 1

# 多行文本输入框
@export_multiline var description: String = ""

# 分组（在 Inspector 中显示分隔线和标题）
@export_group("战斗属性")
@export var attack: int = 50
@export var defense: int = 30

@export_group("移动属性")
@export var move_range: int = 4
@export var attack_range: int = 1

# 子分组
@export_subgroup("基础属性")
@export var max_hp: int = 100
```

### 文件命名建议

```
res://resources/
├── units/
│   ├── 关羽.tres
│   ├── 张飞.tres
│   └── 诸葛亮.tres
├── skills/
│   ├── 青龙偃月.tres
│   └── 空城计.tres
└── items/
    └── 丹药.tres
```

---

## 6. 在代码中加载和使用

### 方式一：load()（同步加载）

```gdscript
# 运行时加载
var unit_data = load("res://resources/units/关羽.tres") as UnitData

if unit_data:
    print(unit_data.unit_name)  # 关羽
    print(unit_data.attack)     # 85
```

### 方式二：preload()（编译时预加载，推荐）

```gdscript
# 场景加载时就会载入，路径必须是字符串字面量（不能是变量）
const GUANYU_DATA = preload("res://resources/units/关羽.tres")

func _ready():
    print(GUANYU_DATA.unit_name)
```

### 方式三：@export 在 Inspector 中拖入（最常用）

```gdscript
# Unit.gd
extends Node2D

@export var data: UnitData  # 在编辑器 Inspector 里拖入对应的 .tres 文件

func _ready():
    print(data.unit_name)
    print(data.max_hp)
```

### 方式四：动态批量加载

```gdscript
# 加载一个目录下的所有武将数据
func load_all_units() -> Array[UnitData]:
    var units: Array[UnitData] = []
    var dir = DirAccess.open("res://resources/units/")

    if dir:
        dir.list_dir_begin()
        var file_name = dir.get_next()
        while file_name != "":
            if file_name.ends_with(".tres"):
                var path = "res://resources/units/" + file_name
                var data = load(path) as UnitData
                if data:
                    units.append(data)
            file_name = dir.get_next()

    return units
```

### 运行时修改与保存

```gdscript
# 修改数据（注意：这会影响所有引用该资源的地方）
var data = load("res://resources/units/关羽.tres") as UnitData
data.attack = 99

# 保存回文件（开发工具/存档功能）
ResourceSaver.save(data, "res://resources/units/关羽.tres")

# 保存到新路径（不覆盖原文件）
ResourceSaver.save(data, "user://saves/关羽_强化.tres")
```

> `user://` 是用户数据目录，适合存储存档。`res://` 在发布后是只读的。

---

## 7. 常用内置 Resource 类型

Godot 自带很多 Resource 子类，了解它们有助于理解 .tres 的用途：

```gdscript
# 图片
var tex: Texture2D = load("res://icon.png")

# 音频
var audio: AudioStream = load("res://sfx/sword.ogg")

# 字体
var font: Font = load("res://fonts/msyh.ttf")

# 场景（PackedScene 也是 Resource）
var scene: PackedScene = load("res://scenes/Unit.tscn")
var instance = scene.instantiate()

# TileSet（地图格子配置）
var tileset: TileSet = load("res://tilemaps/battle_tiles.tres")

# Shader Material
var mat: ShaderMaterial = load("res://materials/outline.tres")
```

---

## 8. Resource 嵌套与引用

Resource 可以嵌套，一个武将可以包含多个技能 Resource。

### 定义嵌套结构

```gdscript
# skill_data.gd
class_name SkillData
extends Resource

@export var skill_name: String = ""
@export var damage_multiplier: float = 1.5
@export var range: int = 1
@export var cooldown: int = 2
@export_multiline var description: String = ""
```

```gdscript
# unit_data.gd
class_name UnitData
extends Resource

@export var unit_name: String = ""
@export var max_hp: int = 100
@export var attack: int = 50

# 嵌套 Resource 数组
@export var skills: Array[SkillData] = []

# 单个嵌套 Resource
@export var passive_skill: SkillData
```

### .tres 文件中的嵌套表示

**方式一：内嵌子资源（sub_resource）**

技能数据直接写在武将 .tres 文件内部：

```ini
[gd_resource type="UnitData" format=3]

[sub_resource type="SkillData" id="skill_001"]
skill_name = "青龙偃月"
damage_multiplier = 2.0
range = 1
cooldown = 3

[sub_resource type="SkillData" id="skill_002"]
skill_name = "武圣"
damage_multiplier = 1.0

[resource]
unit_name = "关羽"
max_hp = 120
attack = 85
skills = [SubResource("skill_001")]
passive_skill = SubResource("skill_002")
```

**方式二：外部引用（ext_resource）**

技能数据存为独立的 .tres 文件，武将文件引用它：

```ini
[gd_resource type="UnitData" format=3]

[ext_resource type="SkillData" path="res://resources/skills/青龙偃月.tres" id="1"]
[ext_resource type="SkillData" path="res://resources/skills/武圣.tres" id="2"]

[resource]
unit_name = "关羽"
max_hp = 120
skills = [ExtResource("1")]
passive_skill = ExtResource("2")
```

### 两种方式对比

|          | 内嵌 sub_resource            | 外部 ext_resource              |
| -------- | ---------------------------- | ------------------------------ |
| 文件数量 | 少（都在一个文件）           | 多（每个技能独立文件）         |
| 复用性   | 差（数据不能被其他武将共享） | 好（多个武将可共用同一个技能） |
| 适合场景 | 数据唯一、不共享             | 数据需要被多处引用             |

---

## 9. .tres vs .res 的区别

| 对比项       | .tres（文本）      | .res（二进制）             |
| ------------ | ------------------ | -------------------------- |
| 可读性       | 可用文本编辑器打开 | 乱码，不可读               |
| 文件大小     | 较大               | 较小（约小 30-50%）        |
| Git 版本控制 | 友好，可以 diff    | 不友好，每次都是二进制变更 |
| 加载速度     | 略慢               | 略快                       |
| 适合阶段     | 开发阶段           | 发布阶段                   |

**建议：开发时用 .tres，发布时 Godot 会自动转换为优化格式，无需手动处理。**

---

## 10. .tres vs JSON 的选择

### 什么时候用 .tres

- 数据只在 Godot 项目内部使用
- 需要引用图片、音频等 Godot 资源
- 希望在编辑器 Inspector 中可视化编辑
- 数据有类型约束（不希望填错类型）
- 需要嵌套 Godot 内置类型（Vector2、Color 等）

### 什么时候用 JSON

- 数据需要与服务器或其他程序交换
- 策划用 Excel 导出数据（Excel → JSON → 游戏）
- 需要在游戏外部工具中编辑
- 玩家可见的存档（便于玩家手动修改）

### JSON 加载示例（对比参考）

```gdscript
# 加载 JSON
func load_json(path: String) -> Dictionary:
    var file = FileAccess.open(path, FileAccess.READ)
    var json = JSON.new()
    json.parse(file.get_as_text())
    return json.get_data()

# 使用时需要手动转换类型，没有类型安全
var data = load_json("res://data/units.json")
var attack = data["关羽"]["attack"]  # 可能是 int 也可能是 String，取决于 JSON
```

---

## 11. 实战案例：三国战棋武将系统

### 完整的数据结构设计

```gdscript
# skill_data.gd
class_name SkillData
extends Resource

@export var skill_name: String = ""
@export_multiline var description: String = ""
@export_enum("主动", "被动", "羁绊") var skill_type: String = "主动"
@export var damage_multiplier: float = 1.0
@export var heal_amount: int = 0
@export_range(1, 5) var skill_range: int = 1
@export_range(0, 10) var cooldown: int = 0
@export var icon: Texture2D
```

```gdscript
# bond_data.gd
class_name BondData
extends Resource

@export var bond_name: String = ""
@export var required_units: PackedStringArray = []   # 武将名列表
@export_range(2, 5) var min_count: int = 2           # 至少几人触发
@export_multiline var description: String = ""

@export_group("加成属性")
@export_range(0, 1.0) var attack_bonus: float = 0.0
@export_range(0, 1.0) var defense_bonus: float = 0.0
@export_range(0, 1.0) var hp_bonus: float = 0.0
```

```gdscript
# unit_data.gd
class_name UnitData
extends Resource

@export_group("基本信息")
@export var unit_name: String = ""
@export_enum("魏", "蜀", "吴", "群") var faction: String = "蜀"
@export_enum("步兵", "骑兵", "弓兵", "谋士", "医师") var unit_type: String = "步兵"
@export_range(1, 5) var star_level: int = 3
@export var portrait: Texture2D
@export var battle_sprite: Texture2D

@export_group("战斗属性")
@export var max_hp: int = 100
@export var attack: int = 50
@export var defense: int = 30
@export var speed: int = 10              # 决定行动顺序

@export_group("移动属性")
@export_range(1, 8) var move_range: int = 4
@export_range(1, 5) var attack_range: int = 1

@export_group("技能")
@export var active_skill: SkillData
@export var passive_skill: SkillData

@export_group("成长")
@export var hp_growth: float = 0.1       # 每级成长率
@export var attack_growth: float = 0.08
@export var defense_growth: float = 0.06
```

### 武将管理器

```gdscript
# UnitDatabase.gd — 自动加载 (AutoLoad)
extends Node

var all_units: Dictionary = {}   # unit_name -> UnitData
var all_bonds: Array[BondData] = []

func _ready():
    _load_all_units()
    _load_all_bonds()

func _load_all_units():
    var dir = DirAccess.open("res://resources/units/")
    if not dir:
        return
    dir.list_dir_begin()
    var file_name = dir.get_next()
    while file_name != "":
        if file_name.ends_with(".tres"):
            var data = load("res://resources/units/" + file_name) as UnitData
            if data:
                all_units[data.unit_name] = data
        file_name = dir.get_next()

func get_unit(name: String) -> UnitData:
    return all_units.get(name, null)

func get_units_by_faction(faction: String) -> Array[UnitData]:
    return all_units.values().filter(func(u): return u.faction == faction)
```

### 战斗中使用武将数据

```gdscript
# Unit.gd — 战场上的武将节点
extends Node2D

@export var data: UnitData      # 在编辑器拖入 .tres 文件

var current_hp: int
var grid_pos: Vector2i
var has_acted: bool = false

func _ready():
    # 用 Resource 数据初始化战斗状态
    current_hp = data.max_hp
    $Sprite2D.texture = data.battle_sprite
    $NameLabel.text = data.unit_name

func take_damage(amount: int):
    var actual_damage = max(1, amount - data.defense)
    current_hp -= actual_damage
    if current_hp <= 0:
        die()

func get_attack_power() -> int:
    return data.attack

func can_move_to(target_pos: Vector2i) -> bool:
    var distance = abs(target_pos.x - grid_pos.x) + abs(target_pos.y - grid_pos.y)
    return distance <= data.move_range
```

### 存档系统

```gdscript
# SaveManager.gd
const SAVE_PATH = "user://saves/"

# 保存武将强化数据（不修改原 .tres，存到用户目录）
func save_unit_progress(unit_data: UnitData, level: int, exp: int):
    var save_data = {
        "unit_name": unit_data.unit_name,
        "level": level,
        "exp": exp,
        "attack": unit_data.attack,
        "max_hp": unit_data.max_hp
    }
    var file = FileAccess.open(SAVE_PATH + unit_data.unit_name + ".json", FileAccess.WRITE)
    file.store_string(JSON.stringify(save_data))

# 强化武将属性（用 duplicate 避免污染原始数据）
func apply_level_up(base_data: UnitData, level: int) -> UnitData:
    var leveled = base_data.duplicate() as UnitData
    leveled.max_hp = int(base_data.max_hp * (1 + base_data.hp_growth * level))
    leveled.attack = int(base_data.attack * (1 + base_data.attack_growth * level))
    leveled.defense = int(base_data.defense * (1 + base_data.defense_growth * level))
    return leveled
```

---

## 12. 常见问题与注意事项

### Q1：修改 .tres 数据后游戏里没有更新？

资源会被 Godot 缓存。重启编辑器，或调用：

```gdscript
# 强制重新加载（开发调试用）
ResourceLoader.load("res://resources/units/关羽.tres", "", ResourceLoader.CACHE_MODE_IGNORE)
```

### Q2：多个 Unit 节点共用一个 .tres，改了一个影响所有的？

这是 Resource 引用共享机制导致的，使用 `duplicate()` 获取独立副本：

```gdscript
func _ready():
    # 错误：所有 Unit 共用同一个 data 对象
    # var runtime_data = data

    # 正确：每个 Unit 独立一份
    var runtime_data = data.duplicate(true)  # true = 深拷贝（连嵌套 Resource 也复制）
```

### Q3：@export 的数组在编辑器里怎么添加元素？

在 Inspector 中，数组字段右边有 `+` 按钮，点击添加元素。也可以直接把 .tres 文件拖入数组槽位。

### Q4：.tres 文件可以在运行时动态创建吗？

可以，直接 `new()` 然后填数据：

```gdscript
var new_unit = UnitData.new()
new_unit.unit_name = "吕布"
new_unit.attack = 100
new_unit.max_hp = 150

# 可选：保存到文件
ResourceSaver.save(new_unit, "res://resources/units/吕布.tres")
```

### Q5：发布游戏后 res:// 路径还能用 load() 吗？

可以，`res://` 在发布后映射到 PCK 包内，`load()` 仍然有效，但**无法写入**。需要写入的数据（存档）必须用 `user://`。

### Q6：能在 .tres 里存自定义方法逻辑吗？

不能在 .tres 文件本身写逻辑，但 Resource 的 GDScript 类（`.gd` 文件）里可以写方法，.tres 只存数据：

```gdscript
# unit_data.gd — 数据 + 方法都在这里
class_name UnitData
extends Resource

@export var attack: int = 50
@export var level: int = 1

# 方法写在 .gd 里，.tres 只保存 @export 的字段值
func get_scaled_attack() -> int:
    return int(attack * (1.0 + level * 0.1))
```

---

## 总结

| 要点     | 说明                                            |
| -------- | ----------------------------------------------- |
| 定义类   | 继承 Resource，加 class_name，字段用 @export    |
| 创建实例 | 编辑器右键 New Resource，或代码 `.new()`        |
| 加载     | `preload()`（编译期）或 `load()`（运行时）      |
| 共享     | 同路径只有一份，需独立副本用 `.duplicate(true)` |
| 存档     | 原始数据存 res://，运行时修改存 user://         |
| 嵌套     | 用 Array[YourResource] 实现数据嵌套             |
| 格式选择 | 开发用 .tres（可读），发布后自动优化            |
