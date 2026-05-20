# Godot 2D Navigation

## 1. 导航系统概览

### 三种方案对比

| 方案                 | 原理               | 适用场景             | 复杂度 |
| -------------------- | ------------------ | -------------------- | ------ |
| `NavigationRegion2D` | 导航网格，自动避障 | 自由移动的 2D 场景   | 中     |
| `AStarGrid2D`        | 网格 A\*，逐格寻路 | Tilemap 像素游戏     | 低     |
| `AStar2D`            | 自定义节点图       | 路点系统、跨地图连接 | 高     |

**像素 RPG 推荐组合：**

- 地图内寻路 → `AStarGrid2D`
- 跨地图连接 → `AStar2D`
- 复杂自由移动场景 → `NavigationRegion2D`

### Godot 4 导航架构

```
NavigationServer2D（单例，全局管理）
├── NavigationMap（每个场景一张地图）
│   ├── NavigationRegion2D（定义可行走区域）
│   └── NavigationObstacle2D（动态障碍）
└── NavigationAgent2D（挂在 NPC 上，查询路径）
```

`AStarGrid2D` 和 `AStar2D` 是独立的算法类，不走 NavigationServer2D，完全自己管理。

---

## 2. NavigationRegion2D + NavigationAgent2D

适合地形复杂、NPC 需要绕过不规则障碍物的场景。

### 场景搭建

```
Node2D（场景根节点）
├── NavigationRegion2D     ← 定义可行走区域
│   └── （在 Inspector 绘制 NavigationPolygon）
├── TileMap
└── NPC（CharacterBody2D）
    └── NavigationAgent2D  ← 挂在 NPC 上
```

### NavigationRegion2D 配置

在 Inspector 里：

1. 新建 `NavigationPolygon`
2. 点击 `NavigationPolygon` → 在视口绘制可行走区域轮廓
3. 障碍物区域用内部多边形挖空

也可以用 TileMap 自动生成（见第 5 节）。

### NavigationAgent2D 配置

| 属性                      | 说明                 | 推荐值（像素游戏） |
| ------------------------- | -------------------- | ------------------ |
| `path_desired_distance`   | 到路径点的到达阈值   | 4.0                |
| `target_desired_distance` | 到终点的到达阈值     | 8.0                |
| `max_speed`               | 最大速度（影响避障） | 与 NPC 速度一致    |
| `avoidance_enabled`       | 是否开启实时避障     | 看需求             |

### NPC 代码

```gdscript
class_name NPC extends CharacterBody2D

@onready var nav_agent: NavigationAgent2D = $NavigationAgent2D

var speed := 80.0
var target_pos: Vector2

func move_to(pos: Vector2):
    target_pos = pos
    nav_agent.target_position = pos

func _physics_process(delta):
    if nav_agent.is_navigation_finished():
        return

    var next_pos = nav_agent.get_next_path_position()
    var direction = (next_pos - global_position).normalized()

    velocity = direction * speed
    move_and_slide()

    update_facing(velocity)

func update_facing(vel: Vector2):
    if vel.length() < 1.0:
        return
    if abs(vel.x) > abs(vel.y):
        $AnimatedSprite2D.play("walk_right" if vel.x > 0 else "walk_left")
    else:
        $AnimatedSprite2D.play("walk_down" if vel.y > 0 else "walk_up")

func _ready():
    nav_agent.navigation_finished.connect(_on_navigation_finished)
    nav_agent.waypoint_reached.connect(_on_waypoint_reached)

func _on_navigation_finished():
    $AnimatedSprite2D.play("idle")

func _on_waypoint_reached(details: Dictionary):
    pass  # 经过中间路径点时触发
```

### 异步等待到达（配合日程系统）

```gdscript
func travel_and_wait(pos: Vector2) -> void:
    move_to(pos)
    await nav_agent.navigation_finished
    # 到达后继续执行下一步日程
```

---

## 3. AStarGrid2D（Tilemap 寻路）

像素 RPG 的首选方案，与 Tilemap 格子天然对齐。

### 初始化网格

```gdscript
class_name MapNavigation extends Node

var astar := AStarGrid2D.new()
var tilemap: TileMap
var tile_size := Vector2i(16, 16)

func setup(tm: TileMap):
    tilemap = tm
    var rect = tilemap.get_used_rect()

    astar.region = rect
    astar.cell_size = Vector2(tile_size)
    astar.offset = Vector2(tile_size) / 2.0  # 格子中心对齐

    # DIAGONAL_MODE_NEVER                    完全不走斜线（推荐像素游戏）
    # DIAGONAL_MODE_ONLY_IF_NO_OBSTACLES     有障碍时不走斜
    # DIAGONAL_MODE_ALWAYS                   总是走斜线
    astar.diagonal_mode = AStarGrid2D.DIAGONAL_MODE_NEVER

    astar.update()  # 必须调用，应用设置

    mark_obstacles()

func mark_obstacles():
    for cell in tilemap.get_used_cells(0):
        var tile_data = tilemap.get_cell_tile_data(0, cell)
        if tile_data == null:
            continue
        if tile_data.get_custom_data("solid"):
            astar.set_point_solid(cell, true)
```

### 寻路接口

```gdscript
func find_path(from_world: Vector2, to_world: Vector2) -> PackedVector2Array:
    var from_cell = world_to_cell(from_world)
    var to_cell   = world_to_cell(to_world)

    if astar.is_point_solid(to_cell):
        to_cell = find_nearest_walkable(to_cell)

    return astar.get_point_path(from_cell, to_cell)
    # 返回的是世界坐标数组（已经帮你转换好了）

func world_to_cell(world_pos: Vector2) -> Vector2i:
    return tilemap.local_to_map(tilemap.to_local(world_pos))

func cell_to_world(cell: Vector2i) -> Vector2:
    return tilemap.to_global(tilemap.map_to_local(cell))

func find_nearest_walkable(cell: Vector2i) -> Vector2i:
    var directions = [Vector2i(1,0), Vector2i(-1,0), Vector2i(0,1), Vector2i(0,-1)]
    for radius in range(1, 10):
        for d in directions:
            var candidate = cell + d * radius
            if not astar.is_point_solid(candidate):
                return candidate
    return cell
```

### NPC 跟随路径

```gdscript
class_name NPC extends CharacterBody2D

var path: PackedVector2Array = []
var path_index := 0
var speed := 64.0
var current_destination: Vector2

func move_to(world_pos: Vector2):
    current_destination = world_pos
    path = MapNavigation.find_path(global_position, world_pos)
    path_index = 0

func _physics_process(delta):
    if path_index >= path.size():
        velocity = Vector2.ZERO
        return

    var target = path[path_index]
    var diff = target - global_position

    if diff.length() < 2.0:
        path_index += 1
        if path_index >= path.size():
            emit_signal("destination_reached")
        return

    velocity = diff.normalized() * speed
    move_and_slide()
    update_animation(velocity)
```

### 运行时更新障碍

```gdscript
func set_cell_solid(cell: Vector2i, solid: bool):
    astar.set_point_solid(cell, solid)
    for npc in get_tree().get_nodes_in_group("npcs"):
        npc.on_map_changed()
```

### 权重地图（不同地形速度不同）

```gdscript
func mark_terrain_weights():
    for cell in tilemap.get_used_cells(0):
        var tile_data = tilemap.get_cell_tile_data(0, cell)
        var weight = tile_data.get_custom_data("move_cost")
        if weight > 0:
            astar.set_point_weight_scale(cell, weight)
# 泥地 weight=2.0 → NPC 倾向于绕开走
# 道路 weight=0.5 → NPC 优先走道路
```

---

## 4. AStar2D（自定义节点图）

用于路点系统或跨地图连接，手动管理节点。

### 基本概念

```
节点（Point）：有 id 和位置
边（Connection）：连接两个节点，有可选权重
路径：从起点到终点，经过一系列节点
```

### 路点系统

```gdscript
class_name WaypointGraph extends Node

var astar := AStar2D.new()
var waypoints: Dictionary = {}  # name -> id
var next_id := 0

func add_waypoint(name: String, pos: Vector2):
    astar.add_point(next_id, pos)
    waypoints[name] = next_id
    next_id += 1

func connect_waypoints(a: String, b: String, bidirectional := true):
    astar.connect_points(waypoints[a], waypoints[b], bidirectional)

func get_path(from: String, to: String) -> PackedVector2Array:
    return astar.get_point_path(waypoints[from], waypoints[to])

func _ready():
    add_waypoint("entrance", Vector2(100, 200))
    add_waypoint("shop",     Vector2(300, 200))
    add_waypoint("tavern",   Vector2(300, 400))
    add_waypoint("home",     Vector2(100, 400))

    connect_waypoints("entrance", "shop")
    connect_waypoints("shop",     "tavern")
    connect_waypoints("tavern",   "home")
    connect_waypoints("home",     "entrance")
```

### 跨地图地图图

```gdscript
class_name WorldGraph extends Node

var astar := AStar2D.new()

var maps := {
    "town":   0,
    "farm":   1,
    "forest": 2,
    "beach":  3,
    "mine":   4,
}

var exits: Dictionary = {
    "town->farm":    {"from": Vector2(0, 240),   "to": Vector2(800, 240)},
    "farm->town":    {"from": Vector2(800, 240),  "to": Vector2(0, 240)},
    "town->forest":  {"from": Vector2(400, 480),  "to": Vector2(400, 0)},
    "forest->town":  {"from": Vector2(400, 0),    "to": Vector2(400, 480)},
    "forest->beach": {"from": Vector2(800, 300),  "to": Vector2(0, 300)},
    "beach->forest": {"from": Vector2(0, 300),    "to": Vector2(800, 300)},
}

func _ready():
    for map_name in maps:
        astar.add_point(maps[map_name], Vector2.ZERO)

    astar.connect_points(maps["town"],   maps["farm"])
    astar.connect_points(maps["town"],   maps["forest"])
    astar.connect_points(maps["forest"], maps["beach"])
    astar.connect_points(maps["town"],   maps["mine"])

func get_map_route(from_map: String, to_map: String) -> Array[String]:
    var id_path = astar.get_id_path(maps[from_map], maps[to_map])
    var result: Array[String] = []
    for id in id_path:
        result.append(maps.find_key(id))
    return result
    # 例：["town", "forest", "beach"]
```

---

## 5. TileMap 集成

### 在 TileSet 中标记可行走性

1. 打开 TileSet 编辑器
2. 选中 Tile → 自定义数据层 → 添加 `solid`（bool 类型）
3. 不可走的 Tile 勾选 `solid = true`

代码读取：

```gdscript
func is_cell_solid(cell: Vector2i) -> bool:
    var tile_data = tilemap.get_cell_tile_data(0, cell)
    if tile_data == null:
        return true  # 空白格视为不可走
    return tile_data.get_custom_data("solid")
```

### 多层 TileMap

```gdscript
# 地面层（layer 0）：地形，影响寻路
# 装饰层（layer 1）：花草，不影响寻路
# 物体层（layer 2）：箱子、NPC，动态障碍

func mark_obstacles():
    for cell in tilemap.get_used_cells(0):  # 地面层
        if is_cell_solid(cell):
            astar.set_point_solid(cell, true)

    for cell in tilemap.get_used_cells(2):  # 物体层
        astar.set_point_solid(cell, true)
```

### NavigationRegion2D 从 TileMap 自动生成

Godot 4.2+ 支持：

```gdscript
func bake_navigation_from_tilemap(tilemap: TileMap, region: NavigationRegion2D):
    # TileSet 里需要配置 NavigationPolygon
    # 在 TileSet 编辑器 → Physics/Navigation 层 → 为每个 Tile 画导航多边形
    region.bake_navigation_polygon()
    await region.bake_finished
```

---

## 6. 动态障碍物

### NavigationObstacle2D

适合 NavigationRegion2D 方案，自动影响导航网格：

```gdscript
class_name MovingObstacle extends CharacterBody2D

@onready var obstacle: NavigationObstacle2D = $NavigationObstacle2D

func _ready():
    obstacle.avoidance_enabled = true
    obstacle.radius = 16.0
```

### AStarGrid2D 动态障碍

```gdscript
class_name DynamicObstacle extends Node2D

var occupied_cells: Array[Vector2i] = []

func place_at(world_pos: Vector2):
    var cell = tilemap.local_to_map(world_pos)
    occupied_cells.append(cell)
    MapNavigation.astar.set_point_solid(cell, true)
    EventBus.emit_signal("map_changed", cell)

func remove():
    for cell in occupied_cells:
        MapNavigation.astar.set_point_solid(cell, false)
    occupied_cells.clear()
    EventBus.emit_signal("map_changed", Vector2i.ZERO)
```

NPC 响应地图变化：

```gdscript
func _ready():
    EventBus.map_changed.connect(_on_map_changed)

func _on_map_changed(changed_cell: Vector2i):
    if path.is_empty():
        return
    if is_path_blocked():
        move_to(current_destination)

func is_path_blocked() -> bool:
    for point in path:
        var cell = tilemap.local_to_map(point)
        if MapNavigation.astar.is_point_solid(cell):
            return true
    return false
```

---

## 7. 跨地图寻路

### 完整实现

```gdscript
class_name CrossMapTravel extends RefCounted

signal travel_completed
signal map_transition_needed(from_map: String, exit_data: Dictionary)

var npc: NPC
var exit_queue: Array = []

func start(from_map: String, dest_map: String, dest_pos: Vector2):
    exit_queue.clear()

    if from_map == dest_map:
        npc.move_to(dest_pos)
        await npc.destination_reached
        travel_completed.emit()
        return

    var map_route = WorldGraph.get_map_route(from_map, dest_map)

    for i in range(map_route.size() - 1):
        var key = "%s->%s" % [map_route[i], map_route[i+1]]
        exit_queue.append({
            "type": "exit",
            "exit_key": key,
            "exit_data": WorldGraph.exits[key]
        })
    exit_queue.append({
        "type": "destination",
        "pos": dest_pos
    })

    await _execute_next()

func _execute_next():
    if exit_queue.is_empty():
        travel_completed.emit()
        return

    var step = exit_queue[0]

    if step["type"] == "exit":
        var from_pos = step["exit_data"]["from"]
        npc.move_to(from_pos)
        await npc.destination_reached
        exit_queue.pop_front()
        map_transition_needed.emit(step["exit_key"].split("->")[0], step["exit_data"])
        await SceneManager.transition_completed
        await _execute_next()

    elif step["type"] == "destination":
        npc.move_to(step["pos"])
        await npc.destination_reached
        exit_queue.pop_front()
        travel_completed.emit()
```

### 离屏 NPC 直接传送

```gdscript
func update_offscreen_npc(game_time: int):
    var target_node = schedule.get_current_node(game_time)

    if target_node.map != current_map:
        current_map = target_node.map
        global_position = target_node.pos
    else:
        global_position = target_node.pos

    current_anim = target_node.anim

func _physics_process(delta):
    if is_in_player_map():
        process_normal_movement(delta)
    else:
        update_offscreen_npc(GameClock.current_time)
```

---

## 8. 性能优化

### 线程寻路

A\* 路径太长时会卡顿，用线程异步计算：

```gdscript
class_name AsyncPathfinder extends Node

signal path_ready(path: PackedVector2Array)

func find_path_async(from: Vector2i, to: Vector2i):
    var thread = Thread.new()
    thread.start(_thread_find_path.bind(from, to, thread))

func _thread_find_path(from: Vector2i, to: Vector2i, thread: Thread):
    var path = astar.get_point_path(from, to)
    call_deferred("_on_path_found", path, thread)

func _on_path_found(path: PackedVector2Array, thread: Thread):
    thread.wait_to_finish()
    path_ready.emit(path)
```

### 路径缓存

```gdscript
class_name PathCache extends Node

var cache: Dictionary = {}
const CACHE_TTL := 5.0
var cache_times: Dictionary = {}

func get_path(from: Vector2i, to: Vector2i) -> PackedVector2Array:
    var key = "%s|%s" % [from, to]
    var now = Time.get_ticks_msec() / 1000.0

    if key in cache and now - cache_times[key] < CACHE_TTL:
        return cache[key]

    var path = astar.get_point_path(from, to)
    cache[key] = path
    cache_times[key] = now
    return path

func invalidate_all():
    cache.clear()
    cache_times.clear()
```

### NPC 更新频率

```gdscript
var update_interval := 0.1
var update_timer := 0.0

func _physics_process(delta):
    follow_path(delta)  # 移动每帧执行

    update_timer += delta
    if update_timer >= update_interval:
        update_timer = 0.0
        check_schedule()

func get_update_interval() -> float:
    var dist = global_position.distance_to(player.global_position)
    if dist < 200:  return 0.05
    if dist < 500:  return 0.2
    return 1.0
```

---

## 9. 调试工具

### 可视化路径

```gdscript
class_name NavDebug extends Node2D

var npc: NPC

func _draw():
    if not OS.is_debug_build():
        return
    if npc.path.is_empty():
        return

    for i in range(npc.path.size() - 1):
        var a = npc.path[i]   - global_position
        var b = npc.path[i+1] - global_position
        draw_line(a, b, Color.YELLOW, 1.0)

    if npc.path_index < npc.path.size():
        var target = npc.path[npc.path_index] - global_position
        draw_circle(target, 3.0, Color.RED)

func _process(_delta):
    queue_redraw()
```

### 调试面板

```gdscript
func _draw():
    for npc in get_tree().get_nodes_in_group("npcs"):
        var screen_pos = get_viewport().get_camera_2d().world_to_screen(npc.global_position)
        draw_string(font, screen_pos + Vector2(10, -20),
            "%s | %s | %s" % [npc.name, npc.state, npc.current_map],
            HORIZONTAL_ALIGNMENT_LEFT, -1, 10, Color.WHITE)
```

### 内置导航调试

```gdscript
# 运行时显示导航网格
NavigationServer2D.set_debug_enabled(true)
```

或在编辑器中开启：**Project Settings → Debug → Navigation**

---

## 10. 完整 NPC 示例

```gdscript
class_name ScheduledNPC extends CharacterBody2D

signal destination_reached

@export var npc_id: String = "harvey"

@onready var anim: AnimatedSprite2D = $AnimatedSprite2D
@onready var debug_label: Label = $DebugLabel

var speed := 64.0
var path: PackedVector2Array = []
var path_index := 0
var current_destination: Vector2
var current_map: String = "town"
var state: String = "idle"

var stuck_timer := 0.0
var last_position: Vector2
var repath_timer := 0.0

func _ready():
    last_position = global_position
    add_to_group("npcs")

    GameClock.hour_changed.connect(_on_hour_changed)
    EventBus.map_changed.connect(_on_map_changed)

    _on_hour_changed(GameClock.current_hour)

# ── 日程驱动 ───────────────────────────────────────────

func _on_hour_changed(hour: int):
    var node = ScheduleManager.get_node(npc_id, hour)
    if node == null:
        return

    if node.map != current_map:
        if is_in_player_map():
            var travel = CrossMapTravel.new()
            travel.npc = self
            travel.travel_completed.connect(func(): set_state(node.anim))
            travel.start(current_map, node.map, node.pos)
        else:
            current_map = node.map
            global_position = node.pos
    else:
        move_to(node.pos)

# ── 移动 ───────────────────────────────────────────────

func move_to(world_pos: Vector2):
    current_destination = world_pos
    path = MapNavigation.find_path(global_position, world_pos)
    path_index = 0
    set_state("walk")

func _physics_process(delta):
    if not is_in_player_map():
        return

    _follow_path(delta)
    _check_stuck(delta)
    _update_debug()

func _follow_path(delta):
    if path_index >= path.size():
        return

    var target = path[path_index]
    var diff = target - global_position

    if diff.length() < 2.0:
        path_index += 1
        if path_index >= path.size():
            set_state("idle")
            destination_reached.emit()
        return

    velocity = diff.normalized() * speed
    move_and_slide()
    _update_anim_direction(velocity)

func _check_stuck(delta):
    repath_timer += delta

    if velocity.length() < 2.0 and path_index < path.size():
        stuck_timer += delta
    else:
        stuck_timer = 0.0
        last_position = global_position

    if stuck_timer > 1.0 and repath_timer > 0.5:
        move_to(current_destination)
        stuck_timer = 0.0
        repath_timer = 0.0

# ── 状态与动画 ─────────────────────────────────────────

func set_state(new_state: String):
    state = new_state
    match state:
        "idle":  anim.play("idle_down")
        "work":  anim.play("work")
        "sit":   anim.play("sit")
        "sleep": anim.play("sleep")

func _update_anim_direction(vel: Vector2):
    if vel.length() < 1.0:
        return
    if abs(vel.x) > abs(vel.y):
        anim.play("walk_right" if vel.x > 0 else "walk_left")
    else:
        anim.play("walk_down" if vel.y > 0 else "walk_up")

# ── 工具 ───────────────────────────────────────────────

func is_in_player_map() -> bool:
    return current_map == SceneManager.current_map

func _on_map_changed(_cell):
    if path.is_empty():
        return
    if _is_path_blocked():
        move_to(current_destination)

func _is_path_blocked() -> bool:
    for point in path:
        var cell = MapNavigation.world_to_cell(point)
        if MapNavigation.astar.is_point_solid(cell):
            return true
    return false

func _update_debug():
    if not OS.is_debug_build():
        return
    debug_label.text = "%s\n%s" % [state, current_map]
```

---

## 快速参考

| 需求           | 方案                                                |
| -------------- | --------------------------------------------------- |
| 单张地图寻路   | `AStarGrid2D.get_point_path()`                      |
| 复杂地形寻路   | `NavigationRegion2D` + `NavigationAgent2D`          |
| 跨地图路线规划 | `AStar2D`（地图图）                                 |
| 动态障碍       | `astar.set_point_solid()` 或 `NavigationObstacle2D` |
| 离屏 NPC       | 直接传送，不寻路                                    |
| 卡住检测       | 速度接近 0 且有路径 → 重新寻路                      |
| 调试           | `NavigationServer2D.set_debug_enabled(true)`        |
