# 主流编程语言垃圾回收算法详解

## 1. 垃圾回收基础概念

### 什么是垃圾

在运行时，**垃圾**是指程序不再能访问到的内存对象。判断一个对象是否为垃圾，本质上是回答：**从程序的根集合（Root Set）出发，是否还能到达这个对象？**

根集合通常包括：

- 栈上的局部变量
- 全局/静态变量
- CPU 寄存器中的引用
- JNI/FFI 引用（对于 JVM 等）

### Stop-The-World（STW）

许多 GC 算法在执行阶段需要暂停所有应用线程，称为 **Stop-The-World** 停顿。STW 保证了 GC 扫描期间堆的一致性，但会引入延迟。现代 GC 的核心目标之一就是尽量缩短 STW 时间。

### GC 的核心权衡

| 指标             | 说明                               |
| ---------------- | ---------------------------------- |
| **吞吐量**       | 应用线程占总 CPU 时间的比例        |
| **延迟**         | 单次 GC 停顿的最大/平均时间        |
| **内存占用**     | GC 自身消耗的额外内存              |
| **停顿可预测性** | 停顿时间是否稳定（对实时系统重要） |

三者通常无法同时最优，是 GC 设计的核心取舍。

---

## 2. 核心算法原理

### 2.1 引用计数（Reference Counting）

每个对象维护一个计数器，记录有多少引用指向它。计数归零时立即释放。

```
对象 A (rc=2) <── 变量 x
              <── 对象 B (rc=1)
```

**优点：**

- 对象死亡时立即回收，内存占用低
- 无需 STW，延迟可预测

**缺点：**

- 无法处理**循环引用**（A → B → A，两者计数永不归零）
- 每次赋值都需要原子地更新计数器，多线程开销大
- 缓存不友好（计数器写操作频繁）

**使用语言：** Python（主 GC）、Swift/ObjC（ARC）、Rust（`Rc<T>`/`Arc<T>` 智能指针）

---

### 2.2 标记-清除（Mark-Sweep）

分两个阶段：

1. **标记阶段**：从根集合出发，深度/广度优先遍历所有可达对象并打标记
2. **清除阶段**：扫描整个堆，回收未被标记的对象

```
根 → A → B → D
         ↓
         C    E（不可达，将被回收）
```

**优点：**

- 能处理循环引用
- 实现相对简单

**缺点：**

- 会产生**内存碎片**
- 清除阶段需要扫描整个堆，开销与堆大小成正比
- 经典实现需要 STW

---

### 2.3 标记-压缩（Mark-Compact）

在标记-清除基础上，增加**压缩阶段**，将存活对象移动到堆的一端，消除碎片。

```
回收前: [A][  ][B][  ][C][  ][D]
回收后: [A][B][C][D][          ]
```

**优点：**

- 无内存碎片
- 分配速度极快（只需移动指针，bump pointer allocation）

**缺点：**

- 需要更新所有指向移动对象的引用，开销更大
- 停顿时间更长

---

### 2.4 复制算法（Copying / Semi-Space）

将堆分为两个等大的半区（From-Space 和 To-Space），每次 GC 将存活对象从 From-Space 复制到 To-Space，然后交换两者角色。

```
From-Space: [A][垃圾][B][垃圾][C]
                  ↓ 复制存活对象
To-Space:   [A][B][C][          ]
```

**优点：**

- 分配速度极快（bump pointer）
- 自动消除碎片
- 只遍历存活对象，存活率低时效率高

**缺点：**

- 内存利用率只有 50%
- 存活对象多时复制开销大

**最适合：** 新生代（大多数对象生命周期短，存活率低）

---

### 2.5 分代假说与分代 GC（Generational GC）

绝大多数对象都**生命周期极短**（"弱分代假说"）。基于此，将堆分为：

- **新生代（Young Generation）**：刚分配的对象，GC 频繁，用复制算法
- **老年代（Old Generation）**：存活多次 GC 的对象，GC 较少，用标记-清除/压缩

对象经过若干次 Minor GC 后晋升到老年代（**对象晋升**，Promotion）。

```
堆结构示意：
┌──────────────────────────────────────────┐
│   新生代（Young Gen）                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ │
│  │  Eden    │ │Survivor 0│ │Survivor 1│ │
│  └──────────┘ └──────────┘ └──────────┘ │
├──────────────────────────────────────────┤
│   老年代（Old Gen）                       │
└──────────────────────────────────────────┘
```

---

### 2.6 三色标记（Tri-color Marking）

用于支持**并发 GC**（GC 线程与应用线程同时运行）。对象被染成三种颜色：

- **白色**：未被访问，GC 结束后白色对象即为垃圾
- **灰色**：已被发现但子对象尚未全部扫描
- **黑色**：自身及所有子对象均已扫描

```
初始：所有对象为白色
根引用的对象变灰 → 扫描灰色对象的子节点 → 子节点变灰，自身变黑
最终：黑色=存活，白色=垃圾
```

**挑战——并发修改问题：** 应用线程在 GC 扫描期间可能修改引用关系，可能导致：

- **漏标（Missed Object）**：存活对象被误当垃圾回收 → 程序崩溃（不可接受）
- **多标（Floating Garbage）**：已死对象未被回收 → 本次 GC 浮动垃圾，下次处理（可接受）

解决漏标的两种屏障技术：

| 屏障                        | 原理                 | 代表 GC         |
| --------------------------- | -------------------- | --------------- |
| **写屏障（Write Barrier）** | 在引用赋值时记录变更 | 大多数并发 GC   |
| **读屏障（Read Barrier）**  | 在读取引用时触发检查 | ZGC（着色指针） |

---

## 3. Java / JVM 垃圾回收

JVM 是 GC 技术最丰富的平台，提供多种可选收集器。

### 3.1 堆结构

```
┌──────────────────────────────────────────────────────┐
│  Young Generation                                    │
│  ┌───────────────┐  ┌────────────┐  ┌────────────┐  │
│  │     Eden      │  │Survivor S0 │  │Survivor S1 │  │
│  └───────────────┘  └────────────┘  └────────────┘  │
├──────────────────────────────────────────────────────┤
│  Old Generation (Tenured)                            │
├──────────────────────────────────────────────────────┤
│  Metaspace（JDK 8+，类元数据，不在堆内）              │
└──────────────────────────────────────────────────────┘
```

### 3.2 Serial GC

- **单线程**，STW
- `-XX:+UseSerialGC`
- 适合单核 CPU 或客户端小程序

### 3.3 Parallel GC（吞吐量优先）

- **多线程**并行执行 GC，但仍 STW
- `-XX:+UseParallelGC`（JDK 8 默认）
- 目标：最大化吞吐量，适合批处理任务

### 3.4 CMS（Concurrent Mark-Sweep，已废弃）

JDK 9 标记废弃，JDK 14 移除。首个主流**并发**收集器。

**执行阶段：**

1. **初始标记（Initial Mark）** — STW，标记 GC Roots 直接关联对象
2. **并发标记（Concurrent Mark）** — 与应用并发，遍历对象图
3. **重新标记（Remark）** — STW，修正并发阶段的变动（增量更新写屏障）
4. **并发清除（Concurrent Sweep）** — 与应用并发，清理垃圾

**缺点：**

- 产生内存碎片（无压缩）
- 并发阶段占用 CPU
- Concurrent Mode Failure（老年代满时退化为 Serial GC）

### 3.5 G1 GC（Garbage-First，JDK 9+ 默认）

**核心创新：将堆划分为大量等大的 Region（默认 1MB~32MB），不再有固定的物理分区。**

```
堆布局（示意，每格为一个 Region）：
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ E │ E │ O │ S │ E │ H │ O │ E │
└───┴───┴───┴───┴───┴───┴───┴───┘
E=Eden  S=Survivor  O=Old  H=Humongous（大对象）
```

**GC 阶段：**

1. **Minor GC（Young-only）** — STW，Eden + Survivor → Survivor / Old
2. **并发标记周期（Concurrent Marking Cycle）**
   - 初始标记（STW）→ 并发根区扫描 → 并发标记 → 重新标记（STW）→ 清理（STW）
3. **Mixed GC** — 回收所有新生代 + 垃圾比例最高的部分老年代 Region

**关键特性：**

- 可预测停顿时间目标（`-XX:MaxGCPauseMillis`，默认 200ms）
- 使用**记忆集（Remembered Set, RSet）** 追踪跨 Region 引用
- 使用**卡表（Card Table）** + 写屏障维护 RSet
- 并发标记使用**SATB（Snapshot At The Beginning）** 写屏障，保证标记一致性

**调优参数：**

```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200       # 停顿目标（毫秒）
-XX:G1HeapRegionSize=16m       # Region 大小
-XX:G1NewSizePercent=5         # 新生代最小比例
-XX:G1MaxNewSizePercent=60     # 新生代最大比例
-XX:InitiatingHeapOccupancyPercent=45  # 触发并发标记的堆占用率
```

### 3.6 ZGC（Z Garbage Collector，JDK 15+ 生产就绪）

**目标：亚毫秒级最大停顿（<1ms），支持 TB 级堆。**

**核心技术——着色指针（Colored Pointers）：**

ZGC 将元数据（标记位、重映射位）直接编码在 64 位指针的高位中，无需单独维护标记位图。

```
64 位指针布局：
┌──────────┬──────┬───────────────────────────┐
│  unused  │ 元数据│        对象地址（42位）     │
│  (18位)  │ (4位)│                           │
└──────────┴──────┴───────────────────────────┘
元数据位：Finalizable | Remapped | Marked1 | Marked0
```

**读屏障（Load Barrier）：** 每次从堆中读取引用时触发，检查指针颜色并按需执行重映射，保证应用线程始终看到最新地址。

**执行阶段（几乎全部并发）：**

1. 初始标记（STW，<1ms）
2. 并发标记
3. 重新标记（STW，<1ms）
4. 并发准备重分配
5. 初始重分配（STW，<1ms）
6. 并发重分配（Relocation）
7. 并发重映射（Remapping）

**调优参数：**

```bash
-XX:+UseZGC
-XX:SoftMaxHeapSize=28g        # 软上限，GC 尝试保持堆在此以下
-XX:ZCollectionInterval=5      # 最大 GC 间隔（秒）
```

### 3.7 Shenandoah GC（Red Hat，JDK 12+）

与 ZGC 类似，目标低延迟，但使用**转发指针（Brooks Forwarding Pointer）** 实现并发压缩：每个对象头部增加一个指针字段，对象移动后指向新地址，读屏障间接寻址。

**与 ZGC 对比：**
| | ZGC | Shenandoah |
|--|-----|------------|
| 实现方式 | 着色指针 + 读屏障 | 转发指针 + 读写屏障 |
| 内存开销 | 低（指针内编码）| 稍高（每对象额外字段）|
| 停顿时间 | 更短（通常 <0.5ms）| 极低（通常 <10ms）|
| 压缩方式 | 并发 | 并发 |

---

## 4. Go 垃圾回收

### 4.1 演进历史

| 版本       | 改进                               |
| ---------- | ---------------------------------- |
| Go 1.0–1.4 | 停止世界（STW）标记-清除           |
| Go 1.5     | 引入并发三色标记，STW < 10ms       |
| Go 1.6     | STW < 5ms                          |
| Go 1.8     | STW 降至 <1ms（消除栈重扫）        |
| Go 1.14+   | 异步抢占，协程可在任意点被 GC 停止 |

### 4.2 算法：并发三色标记-清除

Go 使用**非移动（Non-Moving）** GC，对象地址不会改变，无需读屏障（降低运行时开销）。

**写屏障：混合写屏障（Hybrid Write Barrier，Go 1.8+）**

结合了 Dijkstra 插入屏障和 Yuasa 删除屏障：

- 新写入的指针目标变灰（插入屏障）
- 被覆盖的旧指针目标变灰（删除屏障）

这使得栈上对象无需写屏障（栈扫描在 STW 阶段完成），大幅降低停顿时间。

### 4.3 GC 触发与节奏控制

**触发条件：**

- 堆大小达到上次 GC 后的 `GOGC%` 增长（默认 `GOGC=100`，即堆翻倍）
- 距上次 GC 超过 2 分钟（forcegc goroutine）
- 手动调用 `runtime.GC()`

**`GOGC` 参数：**

```bash
GOGC=100   # 默认：堆增长 100% 时触发（内存占用 vs 吞吐量均衡）
GOGC=off   # 禁用 GC
GOGC=200   # 减少 GC 频率，提升吞吐量但内存占用更高
```

**Go 1.19+ `GOMEMLIMIT`：**

```bash
GOMEMLIMIT=1GiB  # 软内存限制，配合 GOGC 使用，防止 OOM
```

### 4.4 执行流程

```
阶段 1: GC 关闭（mutator 正常运行）
         ↓ 触发 GC
阶段 2: 标记准备（STW，<1ms）
        - 开启写屏障
        - 将根对象加入工作队列
         ↓
阶段 3: 并发标记（与 mutator 并发）
        - GC goroutine 从工作队列取灰色对象，扫描变黑
        - 25% CPU 用于 GC（可通过 GOMAXPROCS 间接影响）
         ↓
阶段 4: 标记终止（STW，<1ms）
        - 关闭写屏障
        - 清理工作
         ↓
阶段 5: 并发清除（与 mutator 并发）
        - 懒惰清除（按需清除，摊销到分配操作中）
```

### 4.5 内存分配器（TCMalloc 变体）

Go 的 GC 效率部分来自其高效分配器：

- **mcache**：每个 P（逻辑处理器）的本地缓存，无锁分配
- **mcentral**：按 size class 组织的全局缓存
- **mheap**：从 OS 获取大块内存，管理 spans

小对象（<32KB）从 mcache 分配，大对象直接从 mheap 分配。

---

## 5. Python 垃圾回收

Python（CPython）使用**引用计数为主、分代 GC 为辅**的混合策略。

### 5.1 引用计数（主要机制）

每个 PyObject 包含 `ob_refcnt` 字段。引用计数归零时立即调用 `tp_dealloc` 析构。

```c
// CPython 对象头（简化）
typedef struct _object {
    Py_ssize_t ob_refcnt;   // 引用计数
    PyTypeObject *ob_type;  // 类型指针
} PyObject;
```

**缺点：** 无法处理循环引用，如：

```python
a = []
b = [a]
a.append(b)
del a, b  # 引用计数均为 1，泄漏！
```

### 5.2 分代 GC（处理循环引用）

`gc` 模块实现三代分代收集，专门针对**容器对象**（list、dict、set、class 实例等）。

```
第 0 代：新对象，阈值 700（默认）
第 1 代：经历 1 次 Minor GC，阈值 10
第 2 代：经历 2 次 Minor GC，阈值 10
```

**触发规则：** 当第 N 代新分配对象数超过阈值，执行第 0~N 代的收集。

**循环引用检测算法：**

1. 复制所有容器的引用计数到 `gc_refs` 字段
2. 遍历容器，对其引用的每个对象将 `gc_refs` 减 1
3. `gc_refs > 0` 的对象有外部引用，标记为可达
4. 从可达对象出发传播可达性
5. 剩余 `gc_refs == 0` 的对象即为循环垃圾

```python
import gc

gc.get_threshold()   # (700, 10, 10)
gc.get_count()       # 各代当前对象数
gc.collect(2)        # 手动触发 Full GC
gc.disable()         # 禁用分代 GC（引用计数仍工作）
```

### 5.3 GIL 与 GC 的关系

CPython 的 **全局解释器锁（GIL）** 保证了引用计数操作的线程安全，但也限制了多线程并发。GC 在持有 GIL 的情况下运行，本质上是 STW。

### 5.4 PyPy 的 GC

PyPy 不使用引用计数，而是基于 **Minimark GC**（分代复制 GC），性能显著优于 CPython，但与 C 扩展的兼容性是其挑战。

---

## 6. JavaScript / V8 垃圾回收

V8（Chrome/Node.js 引擎）是工程化 GC 的典范。

### 6.1 堆结构

```
V8 堆划分：
┌──────────────────────────────────────────────────┐
│  New Space（新生代）                              │
│  ┌───────────────────┐  ┌───────────────────┐   │
│  │     From-Space    │  │     To-Space      │   │
│  └───────────────────┘  └───────────────────┘   │
├──────────────────────────────────────────────────┤
│  Old Space（老年代）                              │
├──────────────────────────────────────────────────┤
│  Large Object Space（大对象，不移动）             │
├──────────────────────────────────────────────────┤
│  Code Space（JIT 编译代码）                       │
├──────────────────────────────────────────────────┤
│  Map Space（对象隐藏类）                          │
└──────────────────────────────────────────────────┘
```

### 6.2 Scavenge（新生代 GC）

使用 **Cheney 算法**（半空间复制）：

1. 从根集合扫描 From-Space 中存活对象
2. 复制到 To-Space，原地留下转发指针
3. 交换 From/To 角色

**晋升条件：** 对象已经历一次 Scavenge，或 To-Space 使用率超过 25%。

**特点：** 极快（通常 <1ms），但内存利用率 50%，新生代默认约 32MB。

### 6.3 Mark-Sweep & Mark-Compact（老年代 GC）

老年代使用**增量标记（Incremental Marking）** + **并发清除/压缩**。

**Orinoco 项目（V8 并发 GC）三大优化：**

#### 并行 GC（Parallel GC）

多个 GC 辅助线程同时工作，主线程也参与，仍需 STW 但停顿更短。

#### 增量标记（Incremental Marking）

将标记工作拆分为许多小步骤，穿插在 JS 执行之间（每步约 1~2ms）。

```
JS 执行 → 小步标记 → JS 执行 → 小步标记 → ... → 最终标记 → 清除
```

使用**三色标记 + 写屏障**保证一致性。

#### 并发标记/清除（Concurrent Marking/Sweeping）

GC 工作线程在后台运行，不阻塞主线程（JS 继续执行）。

```
主线程：[JS][JS][JS][最终STW][JS][JS]
后台线程：[并发标记...........][并发清除...]
```

### 6.4 弱引用与 FinalizationRegistry

ES2021 引入的 `WeakRef` 和 `FinalizationRegistry` 允许观察对象的 GC 行为：

```javascript
let target = { data: "important" };
const ref = new WeakRef(target);
const registry = new FinalizationRegistry((value) => {
  console.log(`对象 ${value} 已被回收`);
});
registry.register(target, "myObject");
target = null; // target 可能被回收
```

---

## 7. C# / .NET 垃圾回收

.NET CLR 的 GC 与 JVM 类似，但有其独特设计。

### 7.1 堆结构与分代

```
.NET 堆：
┌─────────────────────────────────────────┐
│  第 0 代（Gen 0）~256KB，新对象          │
├─────────────────────────────────────────┤
│  第 1 代（Gen 1）~2MB，缓冲层            │
├─────────────────────────────────────────┤
│  第 2 代（Gen 2）无上限，长存对象        │
├─────────────────────────────────────────┤
│  大对象堆（LOH）>85000 字节的对象        │
└─────────────────────────────────────────┘
```

**Gen 1 作为缓冲层的设计：** 避免 Gen 0 存活对象直接晋升到 Gen 2（可减少 Full GC 频率）。

### 7.2 工作站 GC vs 服务器 GC

|           | 工作站 GC  | 服务器 GC         |
| --------- | ---------- | ----------------- |
| 场景      | 客户端应用 | ASP.NET、微服务   |
| GC 线程数 | 1          | 每个逻辑 CPU 一个 |
| 各代堆    | 1 份       | 每个 CPU 一份     |
| 吞吐量    | 一般       | 高                |

```csharp
// 配置服务器 GC（runtimeconfig.json）
{
  "configProperties": {
    "System.GC.Server": true,
    "System.GC.HeapHardLimit": 1073741824  // 1GB 内存上限
  }
}
```

### 7.3 后台 GC

.NET 4.0+ 引入后台 GC（Background GC），Gen 2 的标记阶段可与应用并发执行，Gen 0/1 的收集仍需 STW 但可在后台 GC 期间触发。

### 7.4 固定（Pinning）与 GC 压力

在 P/Invoke 等互操作场景，需要固定对象（`GCHandle.Alloc(..., GCHandleType.Pinned)`）防止 GC 移动它。大量固定对象会阻碍压缩，可改用 `Memory<T>` 和 `MemoryPool<T>` 缓解。

### 7.5 GC 通知与控制

```csharp
// 注册 GC 通知（适合负载均衡场景）
GC.RegisterForFullGCNotification(10, 10);

// 手动触发
GC.Collect(2, GCCollectionMode.Forced, blocking: true, compacting: true);

// 禁止 GC（关键代码段）
GC.TryStartNoGCRegion(1024 * 1024);
// ... 关键代码 ...
GC.EndNoGCRegion();
```

---

## 8. Rust 的内存管理（无 GC）

Rust 通过**所有权系统（Ownership System）** 在编译期实现内存安全，运行时零 GC 开销。

### 8.1 所有权规则

1. 每个值有且仅有一个**所有者**
2. 所有者离开作用域时，值被**自动销毁**（`Drop` trait）
3. 所有权可以**移动（Move）** 或**借用（Borrow）**

```rust
{
    let s = String::from("hello");  // s 是所有者
    // 自动生成等价于 C++ RAII 的析构调用
}   // s 离开作用域，内存自动释放（无 GC、无 malloc/free）
```

### 8.2 借用检查器（Borrow Checker）

编译器在编译期检查：

- 不能同时存在多个可变引用（防数据竞争）
- 引用的生命周期不能超过被引用值（防悬垂指针）

```rust
let mut v = vec![1, 2, 3];
let r1 = &v;       // 不可变借用
let r2 = &v;       // 可同时多个不可变借用，OK
// let r3 = &mut v; // 编译错误：不可变借用存在时不能可变借用
```

### 8.3 智能指针（引用计数）

Rust 提供受控的引用计数用于特殊场景：

- `Rc<T>`：单线程引用计数，循环引用用 `Weak<T>` 打破
- `Arc<T>`：线程安全引用计数（原子操作）

```rust
use std::rc::{Rc, Weak};

let a = Rc::new(5);
let b = Rc::clone(&a);  // 引用计数 +1
println!("{}", Rc::strong_count(&a));  // 2
```

### 8.4 为何 Rust 不需要 GC

| 问题     | Rust 解决方案    |
| -------- | ---------------- |
| 内存泄漏 | 所有权析构       |
| 悬垂指针 | 生命周期检查     |
| 数据竞争 | 借用规则         |
| 循环引用 | `Weak<T>` 弱引用 |

代价是**更陡峭的学习曲线**和编写时的约束感。

---

## 9. Ruby 垃圾回收

### 9.1 历史演进

- **MRI 1.x**：朴素 Mark-Sweep，STW
- **MRI 2.1**：分代 GC（RGenGC）
- **MRI 2.2**：增量 GC（IncrementalGC）
- **MRI 3.x**：继续优化，YJIT 编译器带来新挑战

### 9.2 当前算法

Ruby 使用**分代增量标记-清除**：

- **新生代（Young）**：写屏障保护对象（WB-Protected），Minor GC 快速处理
- **老年代（Old）**：Major GC 处理

**特殊问题：** Ruby 大量 C 扩展难以插入写屏障，因此引入"不受保护"对象（Shady Objects），这些对象被视为老年代，增加了 GC 负担。

### 9.3 调优环境变量

```bash
RUBY_GC_HEAP_INIT_SLOTS=10000         # 初始堆槽位数
RUBY_GC_HEAP_FREE_SLOTS=4096          # GC 后保持的空闲槽位
RUBY_GC_HEAP_GROWTH_FACTOR=1.8        # 堆增长系数
RUBY_GC_MALLOC_LIMIT=16MB             # 触发 GC 的 malloc 量
```

---

## 10. Swift / Objective-C ARC

Swift 和 Objective-C 使用**自动引用计数（Automatic Reference Counting, ARC）**，在编译期插入 `retain`/`release` 调用。

### 10.1 ARC 工作原理

```swift
class Person {
    let name: String
    init(_ name: String) { self.name = name }
    deinit { print("\(name) 被释放") }
}

var p1: Person? = Person("Alice")   // retainCount = 1
var p2 = p1                          // retainCount = 2
p1 = nil                             // retainCount = 1
p2 = nil                             // retainCount = 0 → deinit 调用
```

### 10.2 循环引用与弱引用

ARC 无法自动处理循环引用，需手动标注 `weak` 或 `unowned`：

```swift
class Node {
    var next: Node?           // 强引用，可能循环
    weak var parent: Node?    // 弱引用，不增加计数，可为 nil
    unowned var owner: Owner  // 无主引用，不增加计数，假设始终有效
}
```

|            | `weak`             | `unowned`              |
| ---------- | ------------------ | ---------------------- |
| 引用计数   | 不增加             | 不增加                 |
| 对象释放后 | 自动置 nil         | 悬垂（需确保生命周期） |
| 适用场景   | 不确定对方是否存活 | 确定对方生命周期更长   |

### 10.3 ARC 的优缺点

**优点：**

- 无 STW 停顿，延迟可预测
- 对象释放时机确定，`deinit` 可靠执行

**缺点：**

- 引用计数更新有原子操作开销（Swift 5.7+ 通过"Swift actors"减轻）
- 需要开发者手动处理循环引用
- 性能分析器可以看到大量 `objc_retain`/`objc_release` 调用

---

## 11. 各语言 GC 横向对比

| 语言            | 主要算法         | 最大停顿       | 内存开销        | 循环引用       | 移动对象   |
| --------------- | ---------------- | -------------- | --------------- | -------------- | ---------- |
| Java (G1)       | 分代并发标记压缩 | ~200ms（可调） | 中              | 自动处理       | 是         |
| Java (ZGC)      | 并发着色指针     | <1ms           | 较高            | 自动处理       | 是（并发） |
| Go              | 并发三色标记清除 | <1ms           | 低              | 自动处理       | 否         |
| Python          | 引用计数 + 分代  | 不稳定         | 高（ob_refcnt） | 分代 GC        | 否         |
| JavaScript (V8) | 分代增量并发     | 通常 <10ms     | 中              | 自动处理       | 是         |
| C# (.NET)       | 分代并发压缩     | ~100ms（可调） | 中              | 自动处理       | 是         |
| Rust            | 所有权（无 GC）  | 无停顿         | 最低            | 需手动（Weak） | 不适用     |
| Ruby            | 分代增量标记清除 | ~100ms         | 中              | 自动处理       | 否         |
| Swift           | ARC              | 无停顿         | 低（计数器）    | 需手动（weak） | 否         |

---

## 12. GC 调优通用原则

### 12.1 减少对象分配

GC 的根本压力来自对象分配速率。常见优化：

- **对象池（Object Pool）**：重用频繁创建的对象
- **值类型/栈分配**：尽量使用栈上对象（Go 的逃逸分析、C# struct）
- **减少装箱（Boxing）**：避免将值类型包装为引用类型

### 12.2 控制对象生命周期

- 尽量让对象在新生代死亡（"朝生夕死"），减少晋升到老年代
- 避免长存对象持有短命对象的引用（会阻止 Minor GC 回收）
- 及时置空不再使用的引用（尤其是静态变量、集合中的元素）

### 12.3 诊断工具

| 平台       | 工具                                                    |
| ---------- | ------------------------------------------------------- |
| JVM        | `jstat -gcutil`，`jmap -histo`，JFR，VisualVM，GCViewer |
| Go         | `GODEBUG=gctrace=1`，`pprof`，`runtime/trace`           |
| Python     | `gc` 模块，`tracemalloc`，`objgraph`                    |
| V8/Node.js | `--expose-gc`，`--trace-gc`，Chrome DevTools Memory     |
| .NET       | `dotnet-trace`，`dotnet-counters`，PerfView             |

### 12.4 常见 GC 问题排查

**GC 频率过高：**

- 检查对象分配热点（pprof alloc、async-profiler）
- 增大堆容量或调整 GC 触发阈值

**停顿时间过长：**

- 检查是否有大量存活对象（Full GC / Major GC 慢）
- 切换到低延迟 GC（ZGC、Shenandoah）
- 减少 Finalizer / `__del__` 的使用（延迟回收）

**内存持续增长：**

- 检查内存泄漏：缓存无界增长、事件监听器未注销、全局变量持有引用
- 使用堆快照（heap dump）对比两个时间点的对象增量

---

_文档生成时间：2026-04-30_
