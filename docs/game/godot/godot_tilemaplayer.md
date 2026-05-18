# Godot TileMapLayer

> 适用版本：Godot 4.2+（TileMapLayer 是 4.2 正式拆分的独立节点）

## 1. 基础概念

### TileMapLayer vs 旧版 TileMap

Godot 4.2 之前使用 `TileMap` 节点，内部通过 layer index 管理多层。  
**4.2 起推荐使用 `TileMapLayer`**，每一层是独立节点，更灵活。

| 旧版 TileMap            | 新版 TileMapLayer             |
| ----------------------- | ----------------------------- |
| 单节点多层              | 每层独立节点                  |
| 层之间难以独立控制      | 可单独控制显隐、Z-index、物理 |
| 脚本访问需要 layer 参数 | 直接操作节点本身              |
| Godot 4.2 起标记为过时  | 推荐方式                      |

### 核心概念关系

```
TileMapLayer（地图层节点）
    └── TileSet（瓦片集资源，可共享）
            ├── TileSetSource（瓦片来源）
            │     ├── TileSetAtlasSource（图集来源，最常用）
            │     └── TileSetScenesCollectionSource（场景来源）
            ├── Physics Layer（物理层配置）
            ├── Navigation Layer（导航层配置）
            ├── Occlusion Layer（遮挡层配置）
            └── Custom Data Layer（自定义数据层）
```

### 坐标系

TileMapLayer 使用**网格坐标**，与世界坐标不同。

```gdscript
# 网格坐标 → 世界坐标
var world_pos = tile_map_layer.map_to_local(Vector2i(3, 5))

# 世界坐标 → 网格坐标
var cell = tile_map_layer.local_to_map(Vector2(320, 160))
```

---

## 2. TileSet 瓦片集

TileSet 是 **可复用的资源**，多个 TileMapLayer 可共享同一个 TileSet。

### 2.1 创建 TileSet

**方法一：在编辑器中创建**

1. 选中 TileMapLayer 节点
2. Inspector → TileSet → 新建 TileSet
3. 底部面板出现 TileSet 编辑器

**方法二：单独创建资源文件**

1. FileSystem 面板右键 → New Resource → TileSet
2. 保存为 `my_tileset.tres`
3. 拖入 TileMapLayer 的 TileSet 属性

### 2.2 TileSetAtlasSource（图集来源）

最常用的来源类型，从一张图集图片切割 Tile。

**编辑器操作：**

1. TileSet 面板 → 左下 `+` → Atlas
2. 选择图集图片（Texture）
3. 设置 Tile Size（如 16×16、32×32）
4. 点击 Setup → 自动识别图块

**GDScript 创建：**

```gdscript
var tileset = TileSet.new()
tileset.tile_size = Vector2i(32, 32)

var source = TileSetAtlasSource.new()
source.texture = load("res://art/tileset.png")
source.texture_region_size = Vector2i(32, 32)

# 添加来源，返回 source_id
var source_id = tileset.add_source(source)

# 创建 Tile（图集坐标）
source.create_tile(Vector2i(0, 0))  # 第0列第0行的图块
source.create_tile(Vector2i(1, 0))  # 第1列第0行的图块
```

### 2.3 TileSetScenesCollectionSource（场景来源）

每个 Tile 是一个完整的场景，适合复杂交互对象。

```gdscript
var scene_source = TileSetScenesCollectionSource.new()
var tile_id = scene_source.create_scene_tile(
    load("res://tiles/chest_tile.tscn")
)
tileset.add_source(scene_source)
```

### 2.4 TileSet 全局属性

```gdscript
var tileset = $TileMapLayer.tile_set

tileset.tile_size = Vector2i(16, 16)   # 单个 Tile 尺寸

# Tile 形状
tileset.tile_shape = TileSet.TILE_SHAPE_SQUARE       # 正方形（默认）
tileset.tile_shape = TileSet.TILE_SHAPE_ISOMETRIC    # 等距斜视
tileset.tile_shape = TileSet.TILE_SHAPE_HALF_OFFSET_SQUARE  # 六边形偏移

# 等距地图专用
tileset.tile_layout = TileSet.TILE_LAYOUT_DIAMOND_RIGHT
tileset.tile_offset_axis = TileSet.TILE_OFFSET_AXIS_HORIZONTAL
```

---

## 3. TileMapLayer 属性详解

### Inspector 中的关键属性

```
TileMapLayer
├── Enabled              — 是否启用该层（关闭后不渲染也不处理物理）
├── TileSet              — 使用的 TileSet 资源
├── Y Sort Enabled       — 开启 Y 轴排序（等距地图必备）
├── X Draw Order Reversed— 反转 X 方向绘制顺序
├── Rendering Quadrant Size — 渲染分块大小（影响性能）
└── Collision
      ├── Enabled        — 碰撞总开关
      ├── Use Kinematic Bodies — 用运动体代替静态体
      └── Visibility Mode — 碰撞形状显示模式
```

### GDScript 访问属性

```gdscript
var layer = $TileMapLayer

# 基本
layer.enabled = true
layer.tile_set = load("res://my_tileset.tres")

# Y 排序（等距地图）
layer.y_sort_enabled = true

# 碰撞
layer.collision_enabled = true
layer.collision_visibility_mode = TileMapLayer.DEBUG_VISIBILITY_MODE_FORCE_SHOW

# Z 层级（控制渲染顺序）
layer.z_index = -1  # 背景层放到最后
```

---

## 4. 编辑器使用

### 4.1 绘制工具栏

选中 TileMapLayer 后，底部出现 TileMap 编辑面板：

| 工具   | 快捷键 | 说明                        |
| ------ | ------ | --------------------------- |
| 选择   | S      | 框选 Tile                   |
| 画笔   | B      | 单格绘制                    |
| 线段   | L      | 直线绘制                    |
| 矩形   | R      | 矩形区域填充                |
| 油漆桶 | G      | 洪水填充                    |
| 拾色器 | I      | 拾取已放置的 Tile           |
| 橡皮擦 | E      | 擦除（按住 Ctrl 临时切换）  |
| 随机   | —      | 随机选取当前选中的多个 Tile |

### 4.2 散射（Scatter）

选中多个 Tile 后，右键 → **Use as Random Tiles**，绘制时随机选用，适合草地、石头等自然地形。

### 4.3 图案（Patterns）

可以把一组 Tile 保存为图案，方便复用：

1. 选择工具框选一片区域
2. 右键 → Save as Pattern
3. 切换到 Patterns 标签使用

### 4.4 地形（Terrains）

Terrain 系统可以让相邻 Tile 自动匹配边缘，无需手动选择每个变体。

**配置步骤：**

1. TileSet 面板 → Terrains 标签 → 添加 Terrain Set
2. 设置模式：Match Corners and Sides / Match Corners / Match Sides
3. 为每个 Tile 的各方向绘制 Terrain 归属
4. 编辑器绘制时选用 Terrain 模式，自动处理拼接

**GDScript 使用 Terrain：**

```gdscript
# 用 Terrain 填充矩形区域（自动匹配边缘）
$TileMapLayer.set_cells_terrain_connect(
    cells,           # Array[Vector2i] 要填充的格子列表
    terrain_set,     # int Terrain Set 索引
    terrain,         # int Terrain 索引
    ignore_empty_terrains  # bool
)

# 填充矩形区域
var cells: Array[Vector2i] = []
for x in range(0, 20):
    for y in range(0, 15):
        cells.append(Vector2i(x, y))

$TileMapLayer.set_cells_terrain_connect(cells, 0, 0, true)
```

---

## 5. GDScript 操作 Tile

### 5.1 放置与删除

```gdscript
var layer = $TileMapLayer

# 放置 Tile
# set_cell(coords, source_id, atlas_coords, alternative_tile)
layer.set_cell(Vector2i(3, 5), 0, Vector2i(1, 0))

# 删除 Tile（设为 source_id = -1）
layer.erase_cell(Vector2i(3, 5))

# 或者
layer.set_cell(Vector2i(3, 5), -1)

# 清空整个层
layer.clear()
```

### 5.2 读取 Tile 信息

```gdscript
var cell = Vector2i(3, 5)

# 获取 source_id（-1 表示空）
var source_id = layer.get_cell_source_id(cell)

# 获取图集坐标
var atlas_coords = layer.get_cell_atlas_coords(cell)

# 获取替代 Tile ID（镜像/旋转变体）
var alt_id = layer.get_cell_alternative_tile(cell)

# 判断是否为空
if layer.get_cell_source_id(cell) == -1:
    print("空格子")
```

### 5.3 获取 TileData

TileData 包含该 Tile 的所有配置信息（碰撞、自定义数据等）。

```gdscript
var cell = Vector2i(3, 5)
var tile_data = layer.get_cell_tile_data(cell)

if tile_data:
    # 读取自定义数据
    var is_solid = tile_data.get_custom_data("is_solid")
    var tile_type = tile_data.get_custom_data("type")
    print("is_solid:", is_solid, " type:", tile_type)
```

### 5.4 获取区域内的所有 Tile

```gdscript
# 获取已使用的所有格子坐标
var all_cells = layer.get_used_cells()
for cell in all_cells:
    print(cell)

# 获取特定 source_id 的格子
var grass_cells = layer.get_used_cells_by_id(0, Vector2i(2, 0))

# 获取已使用区域的边界矩形
var rect = layer.get_used_rect()
print("地图范围：", rect)  # Rect2i
```

### 5.5 坐标转换

```gdscript
var layer = $TileMapLayer

# 网格坐标 → 局部坐标（相对于 TileMapLayer 节点）
var local = layer.map_to_local(Vector2i(5, 3))

# 局部坐标 → 网格坐标
var cell = layer.local_to_map(Vector2(160, 96))

# 世界坐标 → 网格坐标（考虑节点自身变换）
var world_pos = get_global_mouse_position()
var local_pos = layer.to_local(world_pos)
var grid_cell = layer.local_to_map(local_pos)

# 获取相邻格子
var neighbors = layer.get_surrounding_cells(Vector2i(5, 3))
for neighbor in neighbors:
    print(neighbor)
```

### 5.6 获取鼠标所在格子

```gdscript
func _process(_delta):
    var mouse_pos = get_global_mouse_position()
    var cell = $TileMapLayer.local_to_map(
        $TileMapLayer.to_local(mouse_pos)
    )
    $HoverLabel.text = "当前格子：%s" % str(cell)
```

---

## 6. 物理碰撞

### 6.1 添加物理层

1. TileSet 面板 → 左侧选 TileSet → Inspector → Physics Layers → 添加
2. 设置 Collision Layer 和 Collision Mask（位掩码）

```gdscript
# 用代码添加物理层
var tileset = $TileMapLayer.tile_set
tileset.add_physics_layer()
tileset.set_physics_layer_collision_layer(0, 1)  # 第0层，Layer 1
tileset.set_physics_layer_collision_mask(0, 1)
```

### 6.2 为 Tile 绘制碰撞形状

在 TileSet 面板：

1. 选中一个 Tile
2. 切换到 Physics 标签
3. 点击 `+` 添加多边形，或使用 "F" 自动生成矩形

```gdscript
# 代码添加碰撞形状
var source = tileset.get_source(0) as TileSetAtlasSource
var tile_data = source.get_tile_data(Vector2i(0, 0), 0)

var polygon = PackedVector2Array([
    Vector2(-16, -16),
    Vector2(16, -16),
    Vector2(16, 16),
    Vector2(-16, 16),
])
tile_data.add_collision_polygon(0)  # 物理层 0
tile_data.set_collision_polygon_points(0, 0, polygon)
```

### 6.3 One-Way 碰撞（单向平台）

```gdscript
var tile_data = source.get_tile_data(Vector2i(2, 0), 0)

# 开启单向碰撞（只能从上方穿越）
tile_data.set_collision_polygon_one_way(0, 0, true)
tile_data.set_collision_polygon_one_way_margin(0, 0, 1.0)
```

### 6.4 运行时检测碰撞层

```gdscript
# 动态修改 TileMapLayer 的碰撞层
$TileMapLayer.collision_layer = 1
$TileMapLayer.collision_mask = 1

# 对特定区域开关碰撞
$TileMapLayer.collision_enabled = false  # 关闭整层碰撞
```

---

## 7. 导航寻路

### 7.1 配置导航层

1. TileSet → Navigation Layers → 添加
2. 为可行走 Tile 绘制导航多边形

```gdscript
# 代码添加导航层
tileset.add_navigation_layer()
```

### 7.2 为 Tile 绘制导航区域

```gdscript
var tile_data = source.get_tile_data(Vector2i(0, 0), 0)

var nav_polygon = NavigationPolygon.new()
var outline = PackedVector2Array([
    Vector2(-16, -16),
    Vector2(16, -16),
    Vector2(16, 16),
    Vector2(-16, 16),
])
nav_polygon.add_outline(outline)
nav_polygon.make_polygons_from_outlines()

tile_data.set_navigation_polygon(0, nav_polygon)
```

### 7.3 配合 NavigationAgent2D 使用

```gdscript
# 场景中需要有 NavigationRegion2D（TileMapLayer 会自动生成）
# 角色挂载 NavigationAgent2D

@onready var nav_agent = $NavigationAgent2D

func move_to(target_pos: Vector2):
    nav_agent.target_position = target_pos

func _physics_process(delta):
    if nav_agent.is_navigation_finished():
        return

    var next_pos = nav_agent.get_next_path_position()
    var direction = (next_pos - global_position).normalized()
    velocity = direction * speed
    move_and_slide()
```

### 7.4 运行时修改导航

```gdscript
# 放置或移除 Tile 后需要通知导航系统更新
# Godot 4 中 TileMapLayer 会自动更新导航，无需手动调用
# 如有延迟问题可用：
await get_tree().physics_frame
```

---

## 8. Tile 动画

### 8.1 在编辑器中设置动画

1. TileSet 面板选中 Tile
2. 右侧出现 Animation 设置
3. 设置 **Columns**（帧数）和 **Speed**（帧率）
4. 还可以设置 **Separation**（帧间距）

### 8.2 动画帧设置方式

```
图集排列示例（4帧水平动画）：
┌────┬────┬────┬────┐
│ F1 │ F2 │ F3 │ F4 │
└────┴────┴────┴────┘

TileSetAtlasSource 设置：
- texture_region_size: (16, 16)  ← 单帧大小
- Tile 的 Animation Columns: 4   ← 横向4帧
- Animation Speed: 8.0           ← 每秒8帧
```

### 8.3 GDScript 控制动画

```gdscript
var source = tileset.get_source(0) as TileSetAtlasSource
var tile_data = source.get_tile_data(Vector2i(0, 0), 0)

# 设置动画列数（每行帧数）
source.set_tile_animation_columns(Vector2i(0, 0), 4)

# 设置动画速度（帧/秒）
source.set_tile_animation_speed(Vector2i(0, 0), 8.0)

# 设置动画模式
source.set_tile_animation_mode(
    Vector2i(0, 0),
    TileSetAtlasSource.TILE_ANIMATION_MODE_DEFAULT  # 或 RANDOM_START_TIMES
)
```

### 8.4 随机起始帧（避免所有动画同步）

```gdscript
source.set_tile_animation_mode(
    Vector2i(0, 0),
    TileSetAtlasSource.TILE_ANIMATION_MODE_RANDOM_START_TIMES
)
```

---

## 9. 自定义数据层

自定义数据可以给每个 Tile 附加任意类型的数据，如伤害值、地形类型、是否可行走等。

### 9.1 添加自定义数据层

1. TileSet Inspector → Custom Data Layers → 添加
2. 设置名称（如 `tile_type`）和类型（String / int / float / bool / Color / Vector2 等）

```gdscript
# 代码添加
tileset.add_custom_data_layer()
tileset.set_custom_data_layer_name(0, "tile_type")
tileset.set_custom_data_layer_type(0, TYPE_STRING)

tileset.add_custom_data_layer()
tileset.set_custom_data_layer_name(1, "damage")
tileset.set_custom_data_layer_type(1, TYPE_INT)
```

### 9.2 为 Tile 设置数据值

```gdscript
var tile_data = source.get_tile_data(Vector2i(3, 0), 0)
tile_data.set_custom_data("tile_type", "lava")
tile_data.set_custom_data("damage", 10)
```

### 9.3 运行时读取自定义数据

```gdscript
func get_tile_info(cell: Vector2i) -> Dictionary:
    var tile_data = $TileMapLayer.get_cell_tile_data(cell)
    if not tile_data:
        return {}
    return {
        "type": tile_data.get_custom_data("tile_type"),
        "damage": tile_data.get_custom_data("damage"),
    }

func _on_player_stepped_on(cell: Vector2i):
    var info = get_tile_info(cell)
    if info.get("type") == "lava":
        player.take_damage(info.get("damage", 0))
```

---

## 10. 地图生成与算法

### 10.1 随机填充地图

```gdscript
func generate_random_map(width: int, height: int):
    var layer = $TileMapLayer
    layer.clear()

    for x in range(width):
        for y in range(height):
            var rand = randf()
            if rand < 0.7:
                # 70% 草地（图集第0列第0行）
                layer.set_cell(Vector2i(x, y), 0, Vector2i(0, 0))
            elif rand < 0.9:
                # 20% 泥土
                layer.set_cell(Vector2i(x, y), 0, Vector2i(1, 0))
            else:
                # 10% 空地
                pass
```

### 10.2 噪声地图（Perlin Noise）

```gdscript
func generate_noise_map(width: int, height: int):
    var layer = $TileMapLayer
    var noise = FastNoiseLite.new()
    noise.seed = randi()
    noise.frequency = 0.05

    for x in range(width):
        for y in range(height):
            var value = noise.get_noise_2d(x, y)  # -1.0 ~ 1.0

            if value > 0.3:
                layer.set_cell(Vector2i(x, y), 0, Vector2i(2, 0))  # 山地
            elif value > -0.1:
                layer.set_cell(Vector2i(x, y), 0, Vector2i(0, 0))  # 草地
            else:
                layer.set_cell(Vector2i(x, y), 0, Vector2i(3, 0))  # 水面
```

### 10.3 洞穴生成（Cellular Automata）

```gdscript
const WIDTH = 40
const HEIGHT = 30
const FILL_PROB = 0.45
const SMOOTH_STEPS = 5

var grid: Array = []

func generate_cave():
    # 初始化随机网格
    grid.resize(WIDTH)
    for x in range(WIDTH):
        grid[x] = []
        for y in range(HEIGHT):
            grid[x].append(1 if randf() < FILL_PROB else 0)

    # 平滑迭代
    for _step in range(SMOOTH_STEPS):
        smooth()

    # 写入 TileMapLayer
    apply_to_tilemap()

func smooth():
    var new_grid = []
    new_grid.resize(WIDTH)
    for x in range(WIDTH):
        new_grid[x] = []
        for y in range(HEIGHT):
            var neighbors = count_solid_neighbors(x, y)
            if grid[x][y] == 1:
                new_grid[x].append(0 if neighbors < 3 else 1)
            else:
                new_grid[x].append(1 if neighbors > 4 else 0)
    grid = new_grid

func count_solid_neighbors(x: int, y: int) -> int:
    var count = 0
    for dx in [-1, 0, 1]:
        for dy in [-1, 0, 1]:
            if dx == 0 and dy == 0:
                continue
            var nx = x + dx
            var ny = y + dy
            if nx < 0 or nx >= WIDTH or ny < 0 or ny >= HEIGHT:
                count += 1  # 边界视为实心
            else:
                count += grid[nx][ny]
    return count

func apply_to_tilemap():
    var layer = $TileMapLayer
    layer.clear()
    for x in range(WIDTH):
        for y in range(HEIGHT):
            if grid[x][y] == 1:
                layer.set_cell(Vector2i(x, y), 0, Vector2i(0, 0))  # 墙
```

### 10.4 基于权重的随机 Tile

```gdscript
const TILE_WEIGHTS = {
    Vector2i(0, 0): 60,  # 草地，权重 60
    Vector2i(1, 0): 25,  # 泥土，权重 25
    Vector2i(2, 0): 10,  # 石头，权重 10
    Vector2i(3, 0): 5,   # 特殊，权重 5
}

func pick_weighted_tile() -> Vector2i:
    var total = 0
    for weight in TILE_WEIGHTS.values():
        total += weight

    var roll = randi() % total
    var cumulative = 0
    for atlas_coord in TILE_WEIGHTS:
        cumulative += TILE_WEIGHTS[atlas_coord]
        if roll < cumulative:
            return atlas_coord

    return Vector2i(0, 0)
```

### 10.5 运行时挖掘/放置（类 Terraria）

```gdscript
func _input(event):
    if event is InputEventMouseButton and event.pressed:
        var mouse_world = get_global_mouse_position()
        var cell = $TileMapLayer.local_to_map(
            $TileMapLayer.to_local(mouse_world)
        )

        if event.button_index == MOUSE_BUTTON_LEFT:
            # 左键放置
            $TileMapLayer.set_cell(cell, 0, Vector2i(0, 0))

        elif event.button_index == MOUSE_BUTTON_RIGHT:
            # 右键挖掘
            $TileMapLayer.erase_cell(cell)
```

---

## 11. 多层地图

### 11.1 场景结构

```
Node2D (World)
├── TileMapLayer (background)   z_index = -2  ← 背景（天空、云）
├── TileMapLayer (terrain)      z_index = -1  ← 地形（地面、墙）
├── TileMapLayer (decoration)   z_index = 0   ← 装饰（草、花）
├── CharacterBody2D (Player)    z_index = 1
└── TileMapLayer (foreground)   z_index = 2   ← 前景（树木遮挡）
```

### 11.2 共享 TileSet

多层共享同一 TileSet，节省内存：

```gdscript
var shared_tileset = load("res://tilesets/world.tres")

$Background.tile_set  = shared_tileset
$Terrain.tile_set     = shared_tileset
$Decoration.tile_set  = shared_tileset
$Foreground.tile_set  = shared_tileset
```

### 11.3 按层管理逻辑

```gdscript
# 只在 terrain 层检测碰撞
func is_solid(world_pos: Vector2) -> bool:
    var cell = $Terrain.local_to_map($Terrain.to_local(world_pos))
    return $Terrain.get_cell_source_id(cell) != -1

# 只在 decoration 层读取装饰数据
func get_decoration(world_pos: Vector2) -> String:
    var cell = $Decoration.local_to_map($Decoration.to_local(world_pos))
    var tile_data = $Decoration.get_cell_tile_data(cell)
    if tile_data:
        return tile_data.get_custom_data("deco_type")
    return ""
```

---

## 12. 性能优化

### 12.1 渲染分块大小

```gdscript
# 默认 16，较大的值减少 draw call，但单次更新开销更大
# 静态地图推荐 64，动态地图推荐 16
$TileMapLayer.rendering_quadrant_size = 64
```

### 12.2 只更新变化的区域

```gdscript
# 避免每帧 clear() + 重绘全图
# 只在格子真正变化时调用 set_cell / erase_cell

var dirty_cells: Array[Vector2i] = []

func mark_dirty(cell: Vector2i):
    if cell not in dirty_cells:
        dirty_cells.append(cell)

func _process(_delta):
    for cell in dirty_cells:
        _update_cell(cell)
    dirty_cells.clear()
```

### 12.3 视野裁剪（大地图）

```gdscript
# 只生成摄像机周围的 Tile，其余懒加载
const LOAD_RADIUS = 20  # 格子数

func update_visible_chunks(camera_cell: Vector2i):
    var visible = Rect2i(
        camera_cell - Vector2i(LOAD_RADIUS, LOAD_RADIUS),
        Vector2i(LOAD_RADIUS * 2, LOAD_RADIUS * 2)
    )
    load_chunk(visible)
    unload_far_chunks(camera_cell)
```

### 12.4 VisibleOnScreenNotifier2D 暂停远处动画

对于含动画 Tile 的层，使用 `VisibleOnScreenEnabler2D` 控制节点处理。

### 12.5 避免频繁读取 TileData

```gdscript
# 差：每帧读取
func _process(_delta):
    var tile_data = layer.get_cell_tile_data(cell)  # 开销较大

# 好：缓存结果
var _tile_type_cache: Dictionary = {}

func get_tile_type(cell: Vector2i) -> String:
    if cell in _tile_type_cache:
        return _tile_type_cache[cell]
    var tile_data = layer.get_cell_tile_data(cell)
    var result = tile_data.get_custom_data("type") if tile_data else ""
    _tile_type_cache[cell] = result
    return result

func invalidate_cache(cell: Vector2i):
    _tile_type_cache.erase(cell)
```

---

## 13. 实战案例

### 13.1 点击格子高亮显示

```gdscript
extends TileMapLayer

var hovered_cell := Vector2i(-9999, -9999)

func _process(_delta):
    var mouse_pos = get_global_mouse_position()
    var cell = local_to_map(to_local(mouse_pos))
    if cell != hovered_cell:
        hovered_cell = cell
        queue_redraw()

func _draw():
    if hovered_cell == Vector2i(-9999, -9999):
        return
    var local_pos = map_to_local(hovered_cell)
    var tile_size = tile_set.tile_size
    var rect = Rect2(
        local_pos - Vector2(tile_size) / 2,
        Vector2(tile_size)
    )
    draw_rect(rect, Color(1, 1, 0, 0.4), true)
    draw_rect(rect, Color(1, 1, 0, 0.9), false, 2.0)
```

### 13.2 保存与加载地图

```gdscript
func save_map(path: String):
    var data = {}
    var layer = $TileMapLayer

    for cell in layer.get_used_cells():
        data["%d,%d" % [cell.x, cell.y]] = {
            "source": layer.get_cell_source_id(cell),
            "atlas": [
                layer.get_cell_atlas_coords(cell).x,
                layer.get_cell_atlas_coords(cell).y
            ],
            "alt": layer.get_cell_alternative_tile(cell)
        }

    var file = FileAccess.open(path, FileAccess.WRITE)
    file.store_string(JSON.stringify(data))
    file.close()

func load_map(path: String):
    var layer = $TileMapLayer
    layer.clear()

    var file = FileAccess.open(path, FileAccess.READ)
    if not file:
        return

    var data = JSON.parse_string(file.get_as_text())
    file.close()

    for key in data:
        var parts = key.split(",")
        var cell = Vector2i(int(parts[0]), int(parts[1]))
        var entry = data[key]
        layer.set_cell(
            cell,
            entry["source"],
            Vector2i(entry["atlas"][0], entry["atlas"][1]),
            entry["alt"]
        )
```

### 13.3 迷雾战争（Fog of War）

```gdscript
extends Node2D

@onready var fog_layer: TileMapLayer = $FogLayer
@onready var map_layer: TileMapLayer = $MapLayer

var revealed_cells: Dictionary = {}

func reveal_around(world_pos: Vector2, radius: int):
    var center = map_layer.local_to_map(map_layer.to_local(world_pos))

    for dx in range(-radius, radius + 1):
        for dy in range(-radius, radius + 1):
            if dx * dx + dy * dy <= radius * radius:
                var cell = center + Vector2i(dx, dy)
                if cell not in revealed_cells:
                    revealed_cells[cell] = true
                    fog_layer.erase_cell(cell)  # 移除迷雾

func fill_fog(rect: Rect2i):
    for x in range(rect.position.x, rect.end.x):
        for y in range(rect.position.y, rect.end.y):
            fog_layer.set_cell(Vector2i(x, y), 0, Vector2i(0, 0))  # 放置迷雾 Tile
```

### 13.4 平台游戏地形检测

```gdscript
extends CharacterBody2D

@onready var terrain: TileMapLayer = $"../Terrain"

func get_tile_below() -> Dictionary:
    # 脚底位置转为格子坐标
    var feet_pos = global_position + Vector2(0, 32)
    var cell = terrain.local_to_map(terrain.to_local(feet_pos))

    var tile_data = terrain.get_cell_tile_data(cell)
    if not tile_data:
        return {}

    return {
        "type": tile_data.get_custom_data("tile_type"),
        "slippery": tile_data.get_custom_data("slippery"),
        "damage": tile_data.get_custom_data("damage"),
    }

func _physics_process(delta):
    if is_on_floor():
        var tile = get_tile_below()
        if tile.get("slippery"):
            # 冰面：减少摩擦力
            velocity.x = lerpf(velocity.x, 0, 0.02)
        else:
            velocity.x = lerpf(velocity.x, 0, 0.2)

        if tile.get("type") == "lava":
            take_damage(tile.get("damage", 1))
```

---

## 14. 常见问题

### Q1：放置 Tile 后碰撞不生效？

```
检查顺序：
1. TileSet → Physics Layers 是否添加了物理层
2. Tile 是否在 TileSet 面板绘制了碰撞形状
3. TileMapLayer → Collision → Enabled 是否勾选
4. TileMapLayer 的 collision_layer / collision_mask 是否匹配角色
```

### Q2：地图修改后导航不更新？

```gdscript
# 等待物理帧后导航会自动重建
await get_tree().physics_frame
# 或强制重建（Godot 4.3+）
NavigationServer2D.bake_from_source_geometry_data(nav_region, source_data)
```

### Q3：如何实现半砖（斜坡）碰撞？

在 TileSet 面板为 Tile 手动绘制三角形碰撞多边形，不要使用自动矩形。

### Q4：TileMapLayer 与 Y 排序

```gdscript
# 等距地图或有高度感的 2D 场景，开启 Y 排序
$TileMapLayer.y_sort_enabled = true

# 同时确保需要参与排序的 Tile 设置了 Y Sort Origin
# TileSet 面板 → 选中 Tile → Y Sort Origin（设置排序基点，通常设在脚底）
```

### Q5：Alternative Tile（替代 Tile）是什么？

```gdscript
# Alternative Tile 是同一图集坐标的镜像/旋转变体
# ID 含义：
# 0 = 原始
# 1 = 水平翻转
# 2 = 垂直翻转
# 3 = 水平+垂直翻转
# 4 = 旋转90°
# ...

layer.set_cell(cell, source_id, atlas_coords, 1)  # 水平镜像版本
```

### Q6：大地图性能差怎么办？

```
优化策略（按优先级）：
1. 增大 rendering_quadrant_size（减少 draw call）
2. 关闭不可见层的 enabled
3. 分区块懒加载（只加载摄像机附近）
4. 避免频繁 clear() 全图重绘
5. 缓存 get_cell_tile_data() 的结果
```

---

## 附录：常用 API 速查

```gdscript
# ===== 写入 =====
layer.set_cell(coord, source_id, atlas_coord, alt_id)
layer.erase_cell(coord)
layer.clear()
layer.set_cells_terrain_connect(cells, terrain_set, terrain)

# ===== 读取 =====
layer.get_cell_source_id(coord)          # -1 = 空
layer.get_cell_atlas_coords(coord)       # Vector2i
layer.get_cell_alternative_tile(coord)   # int
layer.get_cell_tile_data(coord)          # TileData 或 null
layer.get_used_cells()                   # Array[Vector2i]
layer.get_used_cells_by_id(source, atlas)
layer.get_used_rect()                    # Rect2i

# ===== 坐标转换 =====
layer.map_to_local(cell)                 # Vector2i → Vector2
layer.local_to_map(pos)                  # Vector2 → Vector2i
layer.get_surrounding_cells(cell)        # 相邻格子列表

# ===== TileData =====
tile_data.get_custom_data("key")
tile_data.set_custom_data("key", value)
tile_data.get_collision_polygons_count(physics_layer)
tile_data.get_navigation_polygon(nav_layer)
```
