# 830 - xray GUI 框架设计

## 概述

本文档描述 xray 语言的桌面 GUI 框架设计方案。xray 从脚本语言演进到支持 AOT 编译后，已具备构建原生桌面应用的完整能力。本文档定义了 GUI 框架的分层架构、C FFI 机制、声明式 UI 模型、事件系统，以及标志性 Demo —— **Pipeline Monitor**（实时并行数据处理管道监控器）的完整设计。

### 设计目标

1. **原生性能** — AOT 编译的 GUI 代码直接链接 C 渲染库，无 FFI 桥接开销
2. **并发优先** — 协程 + Channel 天然驱动 UI 事件循环和后台任务
3. **声明式 UI** — 利用 Json + type alias 描述组件树，对 xray 开发者零学习成本
4. **跨平台** — 一套代码，macOS/Linux/Windows 三端运行
5. **渐进式** — 从底层 C 绑定到高层声明式框架，分层可独立使用

### 与其他设计文档的关系

| 文档 | 关系 |
|------|------|
| 810 Transpile-to-C | GUI 框架依赖 AOT 编译管线生成原生二进制 |
| 820 ARC 内存管理 | UI 组件树的生命周期由 ARC 管理，Json→struct 优化消除 GC 抖动 |
| 703 类型系统强化 | type alias 为声明式 UI 组件提供类型安全 |

---

## 1. xray 做 GUI 的能力基础

### 1.1 已具备的技术栈

| 能力 | 现状 | GUI 场景价值 |
|------|------|-------------|
| AOT → C → 原生二进制 | 810 设计，Phase 3 已实现 | 直接链接 C GUI 库，零桥接开销 |
| 协程（native-stackless + 独立 VM 栈） | 成熟，M:N 调度 | UI 线程 + 多工作协程并行 |
| Channel（typed, buffered） | 成熟，move 语义 | UI 线程与后台协程的安全通信 |
| Json 对象（Hidden Class） | 成熟，compact mode | 天然的组件属性/状态描述 |
| type alias + 类型推导 | 成熟 | 组件类型定义，编译期检查 |
| 闭包/回调 | 成熟 | 事件处理函数 |
| 模块系统 | 成熟 | UI 库组织为 stdlib 模块 |
| ARC（820 设计） | 设计完成 | UI 组件树零 GC 管理 |

### 1.2 相比其他语言的差异化优势

**协程 = 天然的异步 UI 模型**

传统 GUI 框架中，异步操作需要回调、Promise 链、或显式线程管理。xray 的协程让异步代码看起来像同步代码：

```xray
fn on_search_click(query: string) {
    show_loading()

    // 自动在后台协程执行，不阻塞 UI
    let results = Http.get("https://api.example.com/search?q=" + query)

    // 回到 UI 线程更新
    update_result_list(results)
    hide_loading()
}
```

**Channel = UI 线程安全通信**

多数 GUI 框架需要 `runOnUiThread()`、`Dispatcher.Invoke()`、`SwingUtilities.invokeLater()` 等机制在线程间传递数据。xray 的 Channel 天然解决这个问题：

```xray
let ui_events = Channel<UIEvent>(64)

// 后台协程直接往 Channel 发送更新
go fn() {
    let data = expensive_computation()
    ui_events.send({ type: "update", payload: data })  // move 语义，零拷贝
}()

// UI 主循环非阻塞接收
fn on_frame() {
    while let event = ui_events.try_recv() {
        apply_update(event)
    }
    render()
}
```

**Json + type = 零成本声明式 UI**

xray 的 Json 对象在 820 ARC 设计下可编译为 C struct，声明式 UI 组件零 GC 开销：

```xray
type Button = {
    text: string,
    on_click: fn(),
    style: Style?
}

// 编译后等价于 C struct，栈分配
let btn = Button({ text: "Click", on_click: handler })
```

---

## 2. 分层架构

```
┌─────────────────────────────────────────────────────┐
│  Layer 4: 应用代码                                   │
│  声明式 UI 组件 + 协程业务逻辑                       │
├─────────────────────────────────────────────────────┤
│  Layer 3: xray UI 框架 (stdlib/ui/)                  │
│  Widget 树 · 布局引擎 · 样式系统 · 状态管理           │
├─────────────────────────────────────────────────────┤
│  Layer 2: 平台抽象层 (stdlib/ui/platform/)           │
│  窗口管理 · 输入事件 · 渲染指令 · 剪贴板 · 文件对话框 │
├─────────────────────────────────────────────────────┤
│  Layer 1: C 库绑定 (stdlib/ui/backend/)              │
│  raylib / SDL2 / libui / WebView                     │
├─────────────────────────────────────────────────────┤
│  Layer 0: C FFI 基础设施                              │
│  extern 声明 · 指针类型 · 回调导出 · AOT 链接          │
└─────────────────────────────────────────────────────┘
```

### 2.1 Layer 0: C FFI 基础设施

这是 GUI 框架的前置基础，让 xray AOT 代码能调用任意 C 库。

#### 2.1.1 外部函数声明

```xray
// 方案 A: @extern 装饰器（推荐）
@extern("raylib")
fn InitWindow(width: int, height: int, title: string)

@extern("raylib")
fn WindowShouldClose(): bool

@extern("raylib")
fn BeginDrawing()

@extern("raylib")
fn DrawText(text: string, x: int, y: int, fontSize: int, color: int)
```

AOT 编译器将 `@extern` 函数翻译为 C 函数声明：

```c
// AOT 生成的 C 代码
extern void InitWindow(int width, int height, const char *title);
extern bool WindowShouldClose(void);
extern void BeginDrawing(void);
extern void DrawText(const char *text, int x, int y, int fontSize, Color color);
```

#### 2.1.2 指针类型支持

GUI 库大量使用不透明指针（窗口句柄、纹理、字体等）：

```xray
// 不透明指针：用户不需要知道内部结构
type Texture = opaque     // AOT → void*
type Font = opaque        // AOT → void*
type Window = opaque      // AOT → void*

@extern("raylib")
fn LoadTexture(path: string): Texture

@extern("raylib")
fn DrawTexture(tex: Texture, x: int, y: int, tint: int)
```

AOT 生成的 C 代码中 `opaque` 类型映射为 `void*`，保持类型安全（不同 opaque 类型之间不可互换）。

#### 2.1.3 回调函数导出

C GUI 库需要调用 xray 函数（事件回调、自定义绘制等）：

```xray
// xray 函数可被 C 库作为回调调用
@export
fn my_draw_callback(x: int, y: int) {
    // ...
}

@extern("some_lib")
fn register_callback(cb: fn(int, int))

// 使用
register_callback(my_draw_callback)
```

AOT 编译器将 `@export` 函数生成为标准 C 函数（非 static），并在 `register_callback` 调用处传递函数指针。

#### 2.1.4 类型映射规则

| xray 类型 | C 类型 | 说明 |
|-----------|--------|------|
| `int` | `int64_t` | xray int 固定 64 位 |
| `float` | `double` | xray float 固定 64 位 |
| `bool` | `bool` | C99 _Bool |
| `string` | `const char*` | FFI 边界自动转换 |
| `opaque` | `void*` | 不透明指针 |
| `Array<T>` | - | 不可直接传递，需手动转换 |
| `fn(A): R` | `R(*)(A)` | 函数指针 |

#### 2.1.5 AOT 链接流程

```
xray 源码 (.xr)
    │
    ▼ xray --native --link raylib
┌──────────┐
│ XIR 生成  │
├──────────┤
│ ARC 优化  │  (820 设计)
├──────────┤
│ C 代码生成 │  (810 设计)
│ + extern  │
│ + export  │
├──────────┤
│ C 编译    │  cc -o app output.c -lraylib -lxray_rt
└──────────┘
    │
    ▼
原生二进制 (app)
```

### 2.2 Layer 1: C 库绑定

将底层 C GUI 库封装为 xray 模块。

#### 2.2.1 推荐的渲染后端

| C 库 | 优势 | 劣势 | 适用场景 |
|------|------|------|---------|
| **raylib** | 极简 API（~50 个核心函数），跨平台，自带 2D/3D/音频 | 非原生控件外观 | 快速原型、游戏化 UI、学习用途 |
| **SDL2** | 工业级稳定，广泛使用 | API 较底层，需自绘控件 | 高性能自绘 UI |
| **libui** | 原生控件（GTK/Cocoa/Win32） | 功能较少，项目不太活跃 | 原生外观应用 |
| **sokol** | 极简、单头文件、跨平台 | 比 raylib 更底层 | 嵌入式/极小体积 |

**Phase 1 选择：raylib**

原因：
- 纯 C API，约 50 个核心函数，绑定工作量最小
- 自带窗口管理、输入、2D 绘制、字体渲染，一库搞定
- MIT 许可证，跨 macOS/Linux/Windows
- 适合 Pipeline Monitor demo 的 2D 可视化需求

#### 2.2.2 raylib 绑定示例

```xray
// stdlib/ui/backend/raylib.xr

// ===== Window =====
@extern("raylib") fn InitWindow(width: int, height: int, title: string)
@extern("raylib") fn CloseWindow()
@extern("raylib") fn WindowShouldClose(): bool
@extern("raylib") fn SetTargetFPS(fps: int)

// ===== Drawing =====
@extern("raylib") fn BeginDrawing()
@extern("raylib") fn EndDrawing()
@extern("raylib") fn ClearBackground(color: int)

// ===== Shapes =====
@extern("raylib") fn DrawRectangle(x: int, y: int, w: int, h: int, color: int)
@extern("raylib") fn DrawRectangleRounded(rec: Rectangle, roundness: float,
                                           segments: int, color: int)

// ===== Text =====
@extern("raylib") fn DrawText(text: string, x: int, y: int, size: int, color: int)
@extern("raylib") fn MeasureText(text: string, size: int): int

// ===== Input =====
@extern("raylib") fn IsMouseButtonPressed(button: int): bool
@extern("raylib") fn GetMouseX(): int
@extern("raylib") fn GetMouseY(): int
@extern("raylib") fn IsKeyPressed(key: int): bool

// ===== Colors =====
const RAYWHITE   = 0xF5F5F5FF
const DARKGRAY   = 0x505050FF
const LIGHTGRAY  = 0xC8C8C8FF
const GREEN      = 0x00E436FF
const RED        = 0xFF004DFF
const BLUE       = 0x29ADFFFF
```

### 2.3 Layer 2: 平台抽象层

在 C 库绑定之上提供统一的平台无关接口：

```xray
// stdlib/ui/platform/window.xr

type WindowConfig = {
    title: string,
    width: int,
    height: int,
    fps: int,
    resizable: bool
}

type MouseEvent = {
    x: int,
    y: int,
    button: int,
    action: string    // "press" | "release" | "move"
}

type KeyEvent = {
    key: int,
    action: string    // "press" | "release" | "repeat"
    mods: int         // shift, ctrl, alt bitmask
}

type InputEvent = {
    type: string,        // "mouse" | "key" | "resize" | "close"
    mouse: MouseEvent?,
    key: KeyEvent?
}
```

```xray
// stdlib/ui/platform/canvas.xr

type Color = {
    r: int, g: int, b: int, a: int
}

type Rect = {
    x: int, y: int, width: int, height: int
}

// 平台无关绘图接口
fn canvas_fill_rect(rect: Rect, color: Color)
fn canvas_draw_text(text: string, x: int, y: int, size: int, color: Color)
fn canvas_draw_line(x1: int, y1: int, x2: int, y2: int, color: Color)
fn canvas_measure_text(text: string, size: int): int
fn canvas_fill_rounded_rect(rect: Rect, radius: int, color: Color)
```

### 2.4 Layer 3: xray UI 框架

声明式组件系统，类似 SwiftUI/Flutter 的设计理念，但利用 xray 的 Json + type 实现。

#### 2.4.1 核心 Widget 类型

```xray
// stdlib/ui/widgets.xr

// 基础组件 —— 都是 type alias，AOT 下编译为 C struct

type Text = {
    content: string,
    size: int,
    color: Color?,
    bold: bool
}

type Button = {
    text: string,
    on_click: fn(),
    style: ButtonStyle?,
    disabled: bool
}

type TextInput = {
    value: string,
    placeholder: string,
    on_change: fn(string),
    width: int?
}

type ProgressBar = {
    value: float,        // 0.0 ~ 1.0
    color: Color?,
    height: int
}

type Image = {
    source: string,      // 文件路径或 URL
    width: int?,
    height: int?
}
```

#### 2.4.2 布局组件

```xray
// stdlib/ui/layout.xr

type Column = {
    children: Array<Widget>,
    spacing: int,
    padding: int,
    align: string        // "start" | "center" | "end"
}

type Row = {
    children: Array<Widget>,
    spacing: int,
    padding: int,
    align: string
}

type Stack = {
    children: Array<Widget>   // 层叠布局
}

type Scroll = {
    child: Widget,
    direction: string    // "vertical" | "horizontal" | "both"
}

type Sized = {
    width: int?,
    height: int?,
    child: Widget
}

type Padding = {
    top: int, right: int, bottom: int, left: int,
    child: Widget
}
```

#### 2.4.3 状态管理

```xray
// stdlib/ui/state.xr

// 响应式状态 —— 值变化时自动触发 UI 重绘
type State<T> = {
    value: T,
    on_change: fn(T)?
}

fn State(initial: any): State {
    return {
        value: initial,
        _dirty: false
    }
}

fn State.set(new_value: any) {
    if self.value != new_value {
        self.value = new_value
        self._dirty = true
        if self.on_change {
            self.on_change(new_value)
        }
    }
}
```

#### 2.4.4 App 入口

```xray
// stdlib/ui/app.xr

type AppConfig = {
    title: string,
    width: int,
    height: int,
    fps: int,
    body: fn(): Widget,       // 声明式 UI 构建函数
    on_init: fn()?,           // 初始化回调
    on_tick: fn()?,           // 每帧回调（事件处理、状态更新）
    on_close: fn()?           // 关闭回调
}

fn App(config: AppConfig) {
    // 初始化窗口
    InitWindow(config.width, config.height, config.title)
    SetTargetFPS(config.fps)

    if config.on_init {
        config.on_init()
    }

    // 主循环
    while !WindowShouldClose() {
        // 每帧回调 —— 处理 Channel 消息、更新状态
        if config.on_tick {
            config.on_tick()
        }

        // 构建 Widget 树
        let tree = config.body()

        // 布局计算
        let layout = compute_layout(tree, config.width, config.height)

        // 渲染
        BeginDrawing()
        ClearBackground(RAYWHITE)
        render_tree(layout)
        EndDrawing()
    }

    if config.on_close {
        config.on_close()
    }
    CloseWindow()
}
```

---

## 3. UI 事件模型 —— 协程驱动

### 3.1 事件循环架构

```
┌────────────────────────────────────────────────────┐
│                  UI 主协程                          │
│                                                    │
│  while !should_close {                             │
│      ┌─────────────────────────────────────┐       │
│      │  1. 收集输入事件（鼠标/键盘/窗口）    │       │
│      ├─────────────────────────────────────┤       │
│      │  2. 非阻塞接收所有 Channel 消息       │       │
│      │     while let msg = ch.try_recv()   │       │
│      ├─────────────────────────────────────┤       │
│      │  3. 更新状态（State.set）             │       │
│      ├─────────────────────────────────────┤       │
│      │  4. 重建 Widget 树（如果 dirty）      │       │
│      ├─────────────────────────────────────┤       │
│      │  5. 布局计算                          │       │
│      ├─────────────────────────────────────┤       │
│      │  6. 渲染帧                            │       │
│      └─────────────────────────────────────┘       │
│  }                                                 │
│                                                    │
│  ◄──── Channel ────── 后台协程 1（数据处理）        │
│  ◄──── Channel ────── 后台协程 2（网络请求）        │
│  ◄──── Channel ────── 后台协程 3（文件 I/O）        │
└────────────────────────────────────────────────────┘
```

### 3.2 关键设计：UI 线程永不阻塞

```xray
fn on_tick(state: AppState, events_ch: Channel<AppEvent>) {
    // 非阻塞批量接收 —— 一帧内处理所有待处理消息
    while let event = events_ch.try_recv() {
        match event.type {
            "worker_progress" => state.update_worker(event.worker_id, event.progress)
            "task_complete"   => state.add_result(event.result)
            "error"           => state.add_log("ERROR: " + event.message)
        }
    }
}
```

`try_recv()` 是非阻塞的：有消息就返回，没消息立刻返回 null。这保证了 UI 主循环每帧都能及时渲染，不会因为等待后台数据而卡顿。

### 3.3 后台任务模式

```xray
// 模式 1: Fire-and-forget（启动后台任务，通过 Channel 报告结果）
fn start_download(url: string, result_ch: Channel<DownloadResult>) {
    go fn() {
        let data = Http.get(url)
        result_ch.send({ url: url, data: data, status: "ok" })
    }()
}

// 模式 2: Worker pool（多协程处理任务队列）
fn start_worker_pool(n: int, tasks: Channel<Task>, results: Channel<Result>) {
    for i in 0..n {
        go fn() {
            for task in tasks {          // Channel 迭代，close 后退出
                let result = process(task)
                results.send(result)     // move 语义
            }
        }()
    }
}

// 模式 3: Streaming（持续数据流）
fn start_log_watcher(path: string, logs: Channel<string>) {
    go fn() {
        let f = File.open(path)
        while let line = f.read_line() {
            logs.send(line)
        }
    }()
}
```

---

## 4. Pipeline Monitor —— 标志性 Demo

### 4.1 为什么选择这个 Demo

Pipeline Monitor 同时展示了 xray 的所有差异化能力：

| xray 能力 | 在 Demo 中的体现 |
|-----------|-----------------|
| 协程 | 8+ 个 worker 协程并行处理任务 |
| Channel pipeline | 数据在 input→worker→aggregator→UI 间流动 |
| move 语义 | Channel 传递零拷贝，无数据竞争 |
| 异步 UI | 后台高强度计算，UI 保持 60fps |
| Json + type | 声明式组件 + 状态描述，AOT 下零 GC |
| AOT 性能 | worker 执行 CPU 密集任务接近 C 速度 |
| 并发安全 | 编译期保证，无锁无 race condition |

**一句话描述：10 行代码，8 个并行 worker，1000 个任务，60fps 实时 UI。**

### 4.2 界面设计

```
┌─────────────────────────────────────────────────────────────┐
│  ⚡ xray Pipeline Monitor                        [▶ Run] │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─ Pipeline ──────────────────────────────────────────────┐│
│  │                                                         ││
│  │  ┌────────┐    ┌────────────┐    ┌────────┐   ┌──────┐ ││
│  │  │ Source  │───▶│ Workers ×8 │───▶│ Reduce │──▶│ Sink │ ││
│  │  │ 1000    │    │ ████░░ 62% │    │ ██░░ 25%│  │  156 │ ││
│  │  └────────┘    └────────────┘    └────────┘   └──────┘ ││
│  │                                                         ││
│  └─────────────────────────────────────────────────────────┘│
│                                                             │
│  ┌─ Stats ─────────────────────────────────────────────────┐│
│  │  Throughput: 847 items/s      Active coroutines: 12     ││
│  │  Memory: 4.2 MB (ARC)        GC collections: 0         ││
│  │  ████████████████████░░░░░░░░░  623/1000  (62.3%)       ││
│  └─────────────────────────────────────────────────────────┘│
│                                                             │
│  ┌─ Workers ───────────────────────────────────────────────┐│
│  │  [●] worker-0  running  task#547  ▓▓▓░  78ms           ││
│  │  [●] worker-1  running  task#548  ▓░░░  12ms           ││
│  │  [◌] worker-2  idle     waiting for input               ││
│  │  [●] worker-3  running  task#550  ▓▓░░  45ms           ││
│  │  [●] worker-4  running  task#551  ▓▓▓▓  92ms           ││
│  │  [◌] worker-5  idle     waiting for input               ││
│  │  [●] worker-6  running  task#553  ▓░░░  8ms            ││
│  │  [●] worker-7  running  task#554  ▓▓░░  34ms           ││
│  └─────────────────────────────────────────────────────────┘│
│                                                             │
│  ┌─ Log ───────────────────────────────────────────────────┐│
│  │  12:41:03.456  worker-3 completed task#549 (32ms)       ││
│  │  12:41:03.458  reducer received batch, buffer: 12       ││
│  │  12:41:03.461  worker-0 started task#551                ││
│  │  12:41:03.463  worker-5 completed task#552 (67ms)       ││
│  │  12:41:03.470  reducer flush: 24 results written        ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### 4.3 数据类型定义

```xray
// pipeline_types.xr

type Task = {
    id: int,
    payload: string       // URL / 文件路径 / 数据块
}

type TaskResult = {
    task_id: int,
    data: string,         // 处理后的数据
    duration_ms: int
}

type WorkerStatus = {
    id: int,
    state: string,        // "running" | "idle" | "done"
    current_task: int,    // task id, -1 if idle
    progress: float,      // 0.0 ~ 1.0
    last_duration_ms: int
}

type PipelineStats = {
    total_tasks: int,
    completed: int,
    failed: int,
    items_per_sec: float,
    active_workers: int,
    memory_bytes: int,
    gc_collections: int,
    elapsed_ms: int
}

type LogEntry = {
    timestamp: string,
    level: string,        // "info" | "warn" | "error"
    message: string
}

// UI 更新事件 —— 通过 Channel 从后台发送到 UI 线程
type UIEvent = {
    type: string,         // "worker_update" | "stats_update" | "log" | "complete"
    worker: WorkerStatus?,
    stats: PipelineStats?,
    log: LogEntry?
}
```

在 820 ARC 设计下，这些 type 在 AOT 模式编译为 C struct，所有字段直接内联，零 GC 开销：

```c
// AOT 生成的等价 C 代码
typedef struct {
    int64_t id;
    const char *payload;
} Task;

typedef struct {
    int64_t id;
    int64_t state;       // enum
    int64_t current_task;
    double  progress;
    int64_t last_duration_ms;
} WorkerStatus;
```

### 4.4 核心代码 —— Pipeline 引擎

```xray
// pipeline.xr

import time

const WORKER_COUNT = 8
const TASK_COUNT = 1000
const BUFFER_SIZE = 64

fn run_pipeline(ui_ch: Channel<UIEvent>) {
    let task_ch   = Channel<Task>(BUFFER_SIZE)
    let result_ch = Channel<TaskResult>(BUFFER_SIZE)
    let start_time = Time.now()

    // Stage 1: Source —— 生产任务
    go source(task_ch)

    // Stage 2: Workers —— 并行处理
    for i in 0..WORKER_COUNT {
        go worker(i, task_ch, result_ch, ui_ch)
    }

    // Stage 3: Reducer —— 聚合结果
    go reducer(result_ch, ui_ch, start_time)
}

fn source(output: Channel<Task>) {
    for i in 0..TASK_COUNT {
        output.send({
            id: i,
            payload: "https://example.com/data/" + i.to_string()
        })
    }
    output.close()
}

fn worker(id: int, input: Channel<Task>, output: Channel<TaskResult>,
          ui_ch: Channel<UIEvent>) {

    // Channel 迭代 —— input.close() 后自动退出循环
    for task in input {
        // 通知 UI: 开始处理
        ui_ch.send({
            type: "worker_update",
            worker: {
                id: id,
                state: "running",
                current_task: task.id,
                progress: 0.0,
                last_duration_ms: 0
            }
        })

        // 模拟 CPU 密集处理（AOT 编译后接近 C 性能）
        let start = Time.now()
        let result = process_task(task)
        let elapsed = Time.since_ms(start)

        // 发送结果（move 语义，零拷贝）
        output.send(result)

        // 通知 UI: 完成
        ui_ch.send({
            type: "worker_update",
            worker: {
                id: id,
                state: "idle",
                current_task: -1,
                progress: 1.0,
                last_duration_ms: elapsed
            }
        })

        ui_ch.send({
            type: "log",
            log: {
                timestamp: Time.format_now("HH:mm:ss.SSS"),
                level: "info",
                message: "worker-" + id.to_string() + " completed task#" +
                         task.id.to_string() + " (" + elapsed.to_string() + "ms)"
            }
        })
    }

    // Channel 关闭，worker 退出
    ui_ch.send({
        type: "worker_update",
        worker: { id: id, state: "done", current_task: -1,
                  progress: 1.0, last_duration_ms: 0 }
    })
}

fn reducer(input: Channel<TaskResult>, ui_ch: Channel<UIEvent>, start_time: int) {
    let completed = 0
    let failed = 0
    let batch = []
    let last_report = Time.now()

    for result in input {
        completed = completed + 1
        batch.push(result)

        // 每 100ms 或每 10 条结果，更新统计
        let now = Time.now()
        if batch.length >= 10 || Time.since_ms(last_report) >= 100 {
            let elapsed = Time.since_ms(start_time)
            let speed = if elapsed > 0 {
                (completed * 1000).to_float() / elapsed.to_float()
            } else { 0.0 }

            ui_ch.send({
                type: "stats_update",
                stats: {
                    total_tasks: TASK_COUNT,
                    completed: completed,
                    failed: failed,
                    items_per_sec: speed,
                    active_workers: WORKER_COUNT,
                    memory_bytes: 0,  // TODO: 从 runtime 获取
                    gc_collections: 0,
                    elapsed_ms: elapsed
                }
            })

            batch = []
            last_report = now
        }
    }

    // 管道完成
    ui_ch.send({ type: "complete" })
}

fn process_task(task: Task): TaskResult {
    // 模拟不同耗时的计算任务
    let complexity = task.id % 5
    let iterations = (complexity + 1) * 100000

    // CPU 密集：质数计数（AOT 编译后此循环为原生 C for-loop）
    let count = 0
    for n in 2..iterations {
        let is_prime = true
        for d in 2..(n / 2 + 1) {
            if n % d == 0 {
                is_prime = false
                break
            }
        }
        if is_prime { count = count + 1 }
    }

    return {
        task_id: task.id,
        data: "primes=" + count.to_string(),
        duration_ms: 0  // 由 worker 填充实际耗时
    }
}
```

### 4.5 核心代码 —— GUI 主程序

```xray
// main.xr

import "./ui" as ui                  // App / Column / Row / Text / ProgressBar / Scroll
import "./ui/state" as state         // State
import time

fn main() {
    // 状态
    let stats = State(PipelineStats({
        total_tasks: TASK_COUNT, completed: 0, failed: 0,
        items_per_sec: 0.0, active_workers: 0,
        memory_bytes: 0, gc_collections: 0, elapsed_ms: 0
    }))
    let workers = State(Array<WorkerStatus>.new(WORKER_COUNT))
    let logs = State(Array<LogEntry>.new(0))
    let running = State(false)

    // UI ↔ Pipeline 通信 Channel
    let ui_ch = Channel<UIEvent>(256)

    // 初始化 worker 状态
    for i in 0..WORKER_COUNT {
        workers.value.push({
            id: i, state: "idle", current_task: -1,
            progress: 0.0, last_duration_ms: 0
        })
    }

    App({
        title: "xray Pipeline Monitor",
        width: 900,
        height: 700,
        fps: 60,

        on_init: fn() {
            // 启动 Pipeline
            running.set(true)
            go run_pipeline(ui_ch)
        },

        on_tick: fn() {
            // 非阻塞接收所有待处理事件
            while let event = ui_ch.try_recv() {
                match event.type {
                    "worker_update" => {
                        workers.value[event.worker.id] = event.worker
                        workers.set(workers.value)
                    }
                    "stats_update" => {
                        stats.set(event.stats)
                    }
                    "log" => {
                        logs.value.push(event.log)
                        // 保留最近 100 条
                        if logs.value.length > 100 {
                            logs.value = logs.value.slice(-100)
                        }
                        logs.set(logs.value)
                    }
                    "complete" => {
                        running.set(false)
                    }
                }
            }
        },

        body: fn(): Widget {
            Column({
                spacing: 12,
                padding: 16,
                children: [
                    // ===== Pipeline 可视化 =====
                    pipeline_view(stats.value),

                    // ===== 统计面板 =====
                    stats_panel(stats.value),

                    // ===== Worker 列表 =====
                    worker_list(workers.value),

                    // ===== 日志 =====
                    log_panel(logs.value)
                ]
            })
        }
    })
}
```

### 4.6 核心代码 —— UI 组件

```xray
// components.xr

// ===== Pipeline 可视化 =====
fn pipeline_view(stats: PipelineStats): Widget {
    let progress = if stats.total_tasks > 0 {
        stats.completed.to_float() / stats.total_tasks.to_float()
    } else { 0.0 }

    Panel({
        title: "Pipeline",
        child: Row({
            spacing: 24,
            align: "center",
            children: [
                stage_box("Source", stats.total_tasks.to_string(), BLUE),
                arrow(),
                stage_box("Workers x" + WORKER_COUNT.to_string(),
                          format_percent(progress), GREEN),
                arrow(),
                stage_box("Reduce",
                          format_percent(progress * 0.4), ORANGE),
                arrow(),
                stage_box("Sink", stats.completed.to_string(), PURPLE)
            ]
        })
    })
}

fn stage_box(name: string, label: string, color: Color): Widget {
    Column({
        align: "center",
        children: [
            Rect({ width: 100, height: 60, color: color, radius: 8 }),
            Text({ content: name, size: 14, bold: true }),
            Text({ content: label, size: 12, color: GRAY })
        ]
    })
}

fn arrow(): Widget {
    Text({ content: "───▶", size: 16, color: GRAY })
}

// ===== 统计面板 =====
fn stats_panel(s: PipelineStats): Widget {
    let progress = if s.total_tasks > 0 {
        s.completed.to_float() / s.total_tasks.to_float()
    } else { 0.0 }

    Panel({
        title: "Stats",
        child: Column({
            spacing: 8,
            children: [
                Row({
                    spacing: 40,
                    children: [
                        stat_item("Throughput",
                                  format_float(s.items_per_sec, 1) + " items/s"),
                        stat_item("Active coroutines",
                                  s.active_workers.to_string()),
                        stat_item("Memory",
                                  format_bytes(s.memory_bytes) + " (ARC)"),
                        stat_item("GC collections",
                                  s.gc_collections.to_string())
                    ]
                }),
                ProgressBar({
                    value: progress,
                    height: 20,
                    color: GREEN
                }),
                Text({
                    content: s.completed.to_string() + "/" +
                             s.total_tasks.to_string() + "  (" +
                             format_percent(progress) + ")",
                    size: 12,
                    color: GRAY
                })
            ]
        })
    })
}

fn stat_item(label: string, value: string): Widget {
    Column({
        children: [
            Text({ content: value, size: 18, bold: true }),
            Text({ content: label, size: 11, color: GRAY })
        ]
    })
}

// ===== Worker 列表 =====
fn worker_list(workers: Array<WorkerStatus>): Widget {
    Panel({
        title: "Workers",
        child: Column({
            spacing: 4,
            children: workers.map(fn(w) {
                worker_row(w)
            })
        })
    })
}

fn worker_row(w: WorkerStatus): Widget {
    let icon = if w.state == "running" { "[●]" }
               else if w.state == "idle" { "[◌]" }
               else { "[✓]" }

    let color = if w.state == "running" { GREEN }
                else if w.state == "idle" { GRAY }
                else { BLUE }

    let detail = if w.state == "running" {
        "task#" + w.current_task.to_string() + "  " +
        progress_mini(w.progress) + "  " +
        w.last_duration_ms.to_string() + "ms"
    } else if w.state == "idle" {
        "waiting for input"
    } else {
        "finished"
    }

    Row({
        spacing: 12,
        children: [
            Text({ content: icon, color: color, size: 14 }),
            Text({ content: "worker-" + w.id.to_string(),
                   size: 13, bold: true }),
            Text({ content: w.state, size: 13, color: color }),
            Text({ content: detail, size: 13, color: GRAY })
        ]
    })
}

fn progress_mini(value: float): string {
    let filled = (value * 4).to_int()
    let empty = 4 - filled
    return "▓".repeat(filled) + "░".repeat(empty)
}

// ===== 日志面板 =====
fn log_panel(logs: Array<LogEntry>): Widget {
    Panel({
        title: "Log",
        child: Scroll({
            direction: "vertical",
            child: Column({
                spacing: 2,
                children: logs.slice(-20).map(fn(entry) {
                    Row({
                        spacing: 8,
                        children: [
                            Text({ content: entry.timestamp,
                                   size: 11, color: DARKGRAY }),
                            Text({ content: entry.message,
                                   size: 11, color: LIGHTGRAY })
                        ]
                    })
                })
            })
        })
    })
}

// ===== Panel 容器 =====
fn Panel(config: { title: string, child: Widget }): Widget {
    Column({
        spacing: 4,
        children: [
            Text({ content: config.title, size: 13, bold: true, color: DARKGRAY }),
            Border({
                radius: 6,
                color: BORDER_COLOR,
                child: Padding({
                    top: 8, right: 12, bottom: 8, left: 12,
                    child: config.child
                })
            })
        ]
    })
}
```

### 4.7 数据流全景图

```
                            xray Pipeline Monitor - 数据流

    ┌──────────────────────────── 后台协程 ──────────────────────────┐
    │                                                                │
    │  source()             worker() x 8           reducer()         │
    │  ┌─────────┐         ┌─────────────┐        ┌──────────┐      │
    │  │ 生成     │  Task   │ 处理 + 上报  │ Result │ 聚合统计  │      │
    │  │ 1000 个  │───────▶│ CPU 密集计算 │───────▶│ 计算速度  │      │
    │  │ Task    │ Channel │ 质数计算/... │Channel │ 发送统计  │      │
    │  └─────────┘         └──────┬──────┘        └────┬─────┘      │
    │                             │                     │            │
    │                     UIEvent │ Channel      UIEvent │ Channel   │
    │                             │                     │            │
    └─────────────────────────────┼─────────────────────┼────────────┘
                                  │                     │
                                  ▼                     ▼
    ┌─────────────────── UI 主协程 (60fps) ──────────────────────────┐
    │                                                                │
    │  on_tick():                                                    │
    │    while let event = ui_ch.try_recv()  ◄── 非阻塞批量接收       │
    │      match event.type:                                         │
    │        "worker_update" → 更新 workers State                    │
    │        "stats_update"  → 更新 stats State                      │
    │        "log"           → 追加 logs State                       │
    │        "complete"      → 标记完成                               │
    │                                                                │
    │  body():                                                       │
    │    pipeline_view(stats)  → 管道可视化                           │
    │    stats_panel(stats)    → 统计数字 + 进度条                    │
    │    worker_list(workers)  → 8 个 worker 实时状态                 │
    │    log_panel(logs)       → 滚动日志                             │
    │                                                                │
    │  render() → 60fps 绘制到窗口                                    │
    └────────────────────────────────────────────────────────────────┘
```

### 4.8 运行方式

```bash
# 开发模式（解释执行）
xray run pipeline_monitor.xr

# AOT 编译模式（原生性能）
xray --native --link raylib -o pipeline_monitor pipeline_monitor.xr
./pipeline_monitor

# AOT + ARC 模式（零 GC）
xray --native --nogc --link raylib -o pipeline_monitor pipeline_monitor.xr
./pipeline_monitor
```

### 4.9 性能预期

| 指标 | 解释模式 | AOT 模式 | AOT + ARC |
|------|---------|---------|-----------|
| 启动时间 | ~50ms | ~5ms | ~5ms |
| worker 计算性能 | 1x | 10-50x | 10-50x |
| UI 帧率 | 60fps | 60fps | 60fps |
| 内存 | ~20MB (GC heap) | ~8MB | ~4MB (精确释放) |
| GC 暂停 | 偶发 1-5ms | 偶发 1-5ms | 0ms |
| 二进制大小 | N/A | ~200KB + libraylib | ~150KB + libraylib |

---

## 5. TUI 版本 —— 无需 GUI 库的验证方案

在 GUI 库绑定完成之前，可以用终端 TUI 版本验证核心架构：

```xray
// pipeline_monitor_tui.xr
// 终端版本 —— 纯 xray，无外部依赖

import time

fn main() {
    let ui_ch = Channel<UIEvent>(256)

    // 启动 Pipeline
    go run_pipeline(ui_ch)

    // TUI 主循环 (~30fps)
    let stats = PipelineStats.default()
    let workers = Array<WorkerStatus>.new(WORKER_COUNT)
    let logs = Array<string>.new(0)

    while true {
        // 非阻塞收集所有事件
        let updated = false
        while let event = ui_ch.try_recv() {
            updated = true
            match event.type {
                "worker_update" => workers[event.worker.id] = event.worker
                "stats_update"  => stats = event.stats
                "log"           => {
                    logs.push(event.log.timestamp + "  " + event.log.message)
                    if logs.length > 20 { logs = logs.slice(-20) }
                }
                "complete" => {
                    render_tui(stats, workers, logs)
                    print("\n✅ Pipeline completed!")
                    return
                }
            }
        }

        if updated {
            render_tui(stats, workers, logs)
        }
        Time.sleep(33)  // ~30fps
    }
}

fn render_tui(stats: PipelineStats, workers: Array<WorkerStatus>,
              logs: Array<string>) {
    // ANSI clear screen
    print("\x1b[2J\x1b[H")

    // Header
    print("⚡ xray Pipeline Monitor\n")
    print("─".repeat(60) + "\n")

    // Progress
    let progress = stats.completed.to_float() / stats.total_tasks.to_float()
    let bar_width = 40
    let filled = (progress * bar_width).to_int()
    let bar = "█".repeat(filled) + "░".repeat(bar_width - filled)
    print("[" + bar + "] " + format_percent(progress) + "\n")
    print("  " + stats.completed.to_string() + "/" +
          stats.total_tasks.to_string() +
          "  |  " + format_float(stats.items_per_sec, 1) + " items/s" +
          "  |  " + stats.elapsed_ms.to_string() + "ms\n\n")

    // Workers
    print("Workers:\n")
    for w in workers {
        let icon = if w.state == "running" { "●" } else { "◌" }
        let task_info = if w.current_task >= 0 {
            "task#" + w.current_task.to_string()
        } else { "---" }
        print("  " + icon + " worker-" + w.id.to_string() +
              "  " + w.state.pad_right(8) +
              "  " + task_info + "\n")
    }

    // Logs
    print("\nLog (last 10):\n")
    for entry in logs.slice(-10) {
        print("  " + entry + "\n")
    }
}
```

**TUI 版本的价值**：
- 验证协程 + Channel pipeline 架构
- 验证 UI 主循环 try_recv 模式
- 验证 worker pool 的负载均衡
- 零外部依赖，立即可运行

---

## 6. 实施路线

### Phase 1: C FFI 基础 (2-3 周)

**目标**: AOT 代码能调用外部 C 函数

| 任务 | 说明 | 工作量 |
|------|------|--------|
| `@extern` 语法解析 | Parser 识别 @extern 装饰器 | 2 天 |
| `opaque` 类型支持 | 类型系统新增 opaque → void* | 1 天 |
| XIR extern 函数节点 | XIR 中表示外部函数调用 | 2 天 |
| C codegen extern | xcgen 生成 extern 声明 + 调用 | 2 天 |
| `@export` 支持 | 生成非 static C 函数 | 1 天 |
| `--link` 编译选项 | 链接外部库 (-lraylib 等) | 1 天 |
| string ↔ const char* | FFI 边界自动转换 | 2 天 |
| 测试 | 调用 C math 库函数验证 | 1 天 |

**交付物**: `xray --native --link m test_ffi.xr` 能调用 `cos()`, `sin()` 等 C 函数。

### Phase 2: raylib 绑定 (1-2 周)

**目标**: 能用 xray 代码创建窗口并绘制图形

| 任务 | 说明 | 工作量 |
|------|------|--------|
| raylib 核心绑定 | 窗口/绘制/输入 ~30 个函数 | 3 天 |
| 颜色/矩形类型 | type alias + C struct 映射 | 1 天 |
| 最小 Demo | 窗口 + 矩形 + 文字 + 鼠标交互 | 2 天 |
| 事件循环集成 | 协程调度器 + raylib 主循环共存 | 3 天 |

**交付物**: xray 代码创建窗口，绘制矩形和文字，响应鼠标点击。

**关键挑战**: 协程调度器与 raylib 主循环的集成。raylib 要求在主线程调用 `BeginDrawing()`/`EndDrawing()`，xray 需要在每帧之间让出控制权给协程调度器处理后台任务。

方案：
```c
// AOT 生成的主循环
while (!WindowShouldClose()) {
    // 1. 让协程调度器执行一批后台任务
    xrt_scheduler_tick(max_timeslice_us);

    // 2. UI 协程的 on_tick（处理 Channel 消息）
    xrt_resume_ui_coro();

    // 3. 渲染
    BeginDrawing();
    // ... render ...
    EndDrawing();
}
```

### Phase 3: UI 框架层 (3-4 周)

**目标**: 声明式组件系统可用

| 任务 | 说明 | 工作量 |
|------|------|--------|
| Widget 类型定义 | Text, Button, Row, Column 等 | 2 天 |
| 布局引擎 | 简化版 Flexbox (Row/Column/Padding/Sized) | 5 天 |
| 渲染器 | Widget 树 → raylib 绘制调用 | 3 天 |
| 事件分发 | 点击测试 (hit test) + 冒泡 | 3 天 |
| State 响应式 | 值变化触发重绘 | 2 天 |
| App 入口 | 封装主循环 | 1 天 |
| ProgressBar / Scroll | 常用控件 | 2 天 |

**交付物**: 可用的声明式 UI 框架，支持基本布局和交互。

### Phase 4: Pipeline Monitor Demo (1-2 周)

**目标**: 完整的 Pipeline Monitor 应用

| 任务 | 说明 | 工作量 |
|------|------|--------|
| TUI 版本 | 终端版验证核心逻辑 | 2 天 |
| Pipeline 引擎 | source/worker/reducer + Channel | 2 天 |
| GUI 版本 | 4 个面板 + 实时更新 | 3 天 |
| 视觉优化 | 颜色/动画/圆角/图标 | 2 天 |
| 性能测试 | AOT vs 解释器对比数据 | 1 天 |

**交付物**: 可运行的 Pipeline Monitor，展示 xray 的协程 + Channel + AOT 能力。

### Phase 5: 打磨与扩展 (持续)

- 更多控件：Table, TreeView, Chart, TabBar
- 主题系统：dark/light mode
- 更多渲染后端：SDL2, WebView
- 打包工具：生成 macOS .app / Linux AppImage / Windows .exe

---

## 7. 编译和构建流程

### 7.1 项目结构

```
my_app/
├── main.xr                 # 入口
├── pipeline.xr             # Pipeline 引擎
├── components.xr           # UI 组件
├── pipeline_types.xr       # 类型定义
└── xray.toml               # 项目配置
```

### 7.2 项目配置文件

```toml
# xray.toml
[project]
name = "pipeline-monitor"
version = "0.1.0"
entry = "main.xr"

[build]
mode = "native"             # "run" | "native"
gc = false                  # ARC only, no GC
optimize = true             # -O2

[link]
libraries = ["raylib"]      # C 库链接
include_paths = ["/usr/local/include"]
library_paths = ["/usr/local/lib"]

[ui]
backend = "raylib"          # "raylib" | "sdl2" | "webview"
```

### 7.3 构建命令

```bash
# 开发（解释模式）
xray run main.xr

# 构建原生二进制
xray build

# 等价于
xray --native --nogc --link raylib --opt -o pipeline-monitor main.xr

# 运行
./pipeline-monitor
```

---

## 8. 与同类框架的对比

| 框架 | 语言 | GUI 方式 | 并发模型 | 二进制大小 | 启动时间 |
|------|------|---------|---------|-----------|---------|
| **xray + raylib** | xray | 自绘制 | 协程+Channel | ~200KB+lib | ~5ms |
| Tauri | Rust | WebView | async/await | ~3MB | ~200ms |
| Flutter | Dart | 自绘制 (Skia) | isolate | ~5MB | ~300ms |
| Fyne | Go | 自绘制 (OpenGL) | goroutine+chan | ~10MB | ~50ms |
| Electron | JS | Chromium | event loop | ~100MB | ~1s |
| SwiftUI | Swift | 原生 | GCD | 系统框架 | ~10ms |
| Dear ImGui | C++ | 即时模式 | 手动线程 | ~500KB | ~1ms |

**xray 的定位**: 介于 Dear ImGui（极简、高性能）和 Flutter（声明式、跨平台）之间。比 ImGui 更易用（声明式 + 协程），比 Flutter 更轻量（AOT → C，无 VM）。

---

## 9. 未来扩展

### 9.1 WebView 混合模式

对于需要复杂 UI 的场景（富文本编辑器、复杂表格），可以嵌入 WebView：

```xray
import "./ui/webview" as webview     // 通过 webview.WebView 访问

fn main() {
    let wv = WebView.new({ title: "My App", width: 800, height: 600 })

    // xray ↔ WebView 双向通信
    wv.bind("fetchData", fn(args: string): string {
        // xray 后台处理（协程，不阻塞 WebView）
        let data = Http.get("https://api.example.com/data")
        return data.body
    })

    wv.load_html("<button onclick='window.fetchData()'>Load</button>")
    wv.run()
}
```

### 9.2 自定义渲染组件

高级用户可以直接使用底层 canvas API：

```xray
fn custom_chart(data: Array<float>, width: int, height: int): Widget {
    Canvas({
        width: width,
        height: height,
        on_draw: fn(ctx: DrawContext) {
            let bar_width = width / data.length
            for i in 0..data.length {
                let bar_height = (data[i] * height).to_int()
                ctx.fill_rect({
                    x: i * bar_width,
                    y: height - bar_height,
                    width: bar_width - 2,
                    height: bar_height
                }, GREEN)
            }
        }
    })
}
```

### 9.3 热重载

开发模式下支持代码修改后自动刷新 UI：

```bash
xray watch main.xr   # 文件变化时自动重新执行
```

---

## 10. 总结

xray 从脚本语言演进到具备桌面 GUI 能力，核心优势在于：

1. **AOT → C** — 直接链接 C GUI 库，零 FFI 开销，二进制体积小
2. **协程 + Channel** — 天然解决 GUI 最难的问题（异步不阻塞 UI），代码比回调/Promise 更清晰
3. **Json + type** — 声明式 UI 零学习成本，AOT 下编译为 C struct 零 GC 开销
4. **ARC 内存管理** — UI 组件树精确释放，无 GC 暂停抖动

**Pipeline Monitor 作为标志性 Demo**，用一个应用同时展示了协程并行、Channel 管道、异步 UI、AOT 性能、声明式组件、编译期内存管理六大能力，充分体现了 xray 语言从脚本到系统级应用的跨越。

**实施路径**: Phase 1 (FFI) → Phase 2 (raylib 绑定) → Phase 3 (UI 框架) → Phase 4 (Pipeline Monitor)，总计约 8-11 周可交付完整 Demo。
