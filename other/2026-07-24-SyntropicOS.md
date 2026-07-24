# SyntropicOS：零开销协作式嵌入式 RTOS 框架深度解析

> 📅 日期：2026-07-24  
> 🏷️ 分类：嵌入式 RTOS / 跨平台框架  
> ⭐ Stars：4  
> ✍️ 作者：蔡浩宇（jun-chy）

---

## 项目链接

| 属性 | 信息 |
|------|------|
| **GitHub 地址** | [https://github.com/outlookhazy/SyntropicOS](https://github.com/outlookhazy/SyntropicOS) |
| **作者** | outlookhazy |
| **许可证** | Apache License 2.0 |
| **最近更新** | 2026-07-24 |
| **创建时间** | 2026-06-29 |
| **主要语言** | C (C99) |
| **默认分支** | main |
| **代码大小** | ~7.8 MB |

---

## 项目简介与核心特性

SyntropicOS 是一个零开销、生产级 C99 嵌入式框架，专为深度嵌入式系统设计。它采用**协作式 protothread 协程**实现多任务调度，每个任务仅需 2 字节 RAM（纯 protothread）或 16~28 字节（含调度元数据的完整任务），远低于传统 RTOS（如 FreeRTOS 每线程需 1KB+ 栈空间）。

其核心理念是**"协作式设计"（cooperative-by-design）**——所有 70+ 模块（网络、DSP、电机控制、协议栈、GUI 等）从头开始以显式状态机方式构建，在等待状态（网络握手、硬件 I/O、传感器采样）期间通过 `PT_DEFER` / `PT_WAIT_UNTIL` 将控制权交还调度器，而非阻塞 CPU。

### 核心特性一览

| 特性 | 描述 | 技术细节 |
|------|------|----------|
| **无栈协程** | Protothread 协程，Duff's device 实现 | 2 字节 RAM（`uint16_t lc`） |
| **零堆分配** | 所有状态为调用者拥有或静态分配 | 无 `malloc`/`free`，杜绝内存碎片 |
| **70+ 原生模块** | 驱动、DSP、定点数学、卡尔曼滤波、FOC 电机控制等 | 每个模块可通过 `#define` 独立开关 |
| **工业协议** | SAE J1939、CANopen、Modbus RTU/TCP、NMEA 0183 | 完整协议栈实现 |
| **纯整数数学** | Q16.16 定点库，三角函数、矩阵代数、FFT | 无浮点、无 `libm.a` |
| **DMA 安全** | 裸金属安全 DMA 事务引擎 | 地址对齐验证、D-cache 一致性 |
| **跨平台** | STM32F4（裸金属）、STM32 HAL、ESP32（ESP-IDF）、RP2040（Pico SDK）、Arduino | 统一移植层接口 |
| **Tickless 睡眠** | 支持低功耗 tickless 模式 | 计算最早唤醒时刻，CPU 深度睡眠 |
| **密码学** | BLAKE2s、ChaCha20-Poly1305、X25519 | 纯 C 实现，无外部依赖 |

---

## 硬件架构

### 系统组成图

```
┌─────────────────────────────────────────────────────────────────┐
│                      SyntropicOS 架构                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐    │
│  │ CLI 交互 │  │ 日志系统  │  │ 性能分析  │  │  错误日志     │    │
│  │(syn_cli) │  │(syn_log) │  │(syn_prof)│  │ (syn_errlog) │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘    │
│       │              │              │               │            │
│  ─────┴──────────────┴──────────────┴───────────────┴────        │
│                        应用层 (user tasks)                       │
│  ─────┬──────────────┬──────────────┬───────────────┬────        │
│       │              │              │               │            │
│  ┌────┴─────┐  ┌─────┴────┐  ┌─────┴────┐  ┌──────┴───────┐    │
│  │ 协作调度器 │  │ 定时器轮  │  │ 事件组    │  │  邮箱/消息队列 │    │
│  │(syn_sched)│  │(syn_timer)│ │(syn_event)│  │(syn_mailbox) │    │
│  └────┬─────┘  └─────┬────┘  └─────┬────┘  └──────┬───────┘    │
│       │              │              │               │            │
│  ═════╪══════════════╪══════════════╪═══════════════╪═══        │
│       │     Protothread 协程层 (syn_pt.h)          │            │
│  ═════╪══════════════╪══════════════╪═══════════════╪═══        │
│       │              │              │               │            │
│  ┌────┴────┐  ┌──────┴───┐  ┌───────┴──┐  ┌───────┴──────┐    │
│  │PID 控制 │  │FOC 电机  │  │ DSP 滤波  │  │  网络协议栈    │    │
│  │(syn_pid)│  │(syn_foc) │  │(syn_filter)│ │(MQTT/HTTP/WS)│    │
│  └────┬────┘  └────┬─────┘  └────┬─────┘  └───────┬──────┘    │
│       │            │              │                │            │
│  ┌────┴────────────┴──────────────┴────────────────┴──────┐    │
│  │              驱动层 (GPIO/UART/I2C/SPI/ADC/DMA/CAN)      │    │
│  └────────────────────────┬────────────────────────────────┘    │
│                           │                                     │
│  ┌────────────────────────┴────────────────────────────────┐    │
│  │              移植层 (syn_port_*)                         │    │
│  │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐         │    │
│  │  │ STM32F4│  │  ESP32 │  │ RP2040 │  │ Arduino│         │    │
│  │  │(bare-m)│  │(ESP-IDF)│  │(PicoSDK)│  │ (C++ ) │         │    │
│  │  └────────┘  └────────┘  └────────┘  └────────┘         │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 支持的硬件平台

| 平台 | 移植方式 | 最小 RAM | 典型芯片 |
|------|----------|----------|----------|
| STM32F4 | 裸金属（直接寄存器） | 8 KB | STM32F407VG |
| STM32 HAL | HAL 库封装（跨系列） | 8 KB | STM32F1/F4/L4 |
| ESP32 | ESP-IDF | 32 KB | ESP32/ESP32-S3/ESP32-C3 |
| RP2040 | Pico SDK | 16 KB | RP2040/RP2350 |
| Arduino | C++ SDK | 2 KB | AVR Mega / ARM |
| POSIX | Linux/macOS 仿真 | — | 开发调试用 |

---

## 固件架构

### 文件结构

```
SyntropicOS/
├── src/
│   ├── SyntropicOS.h              # 顶层包含头文件
│   └── syntropic/
│       ├── syntropic.h            # 模块总入口（条件包含各子模块）
│       ├── syn_config_template.h  # 配置模板
│       ├── pt/                    # Protothread 协程核心
│       │   ├── syn_pt.h           # 协程宏定义（Duff's device）
│       │   └── syn_pt_sem.h       # 协程信号量
│       ├── sched/                 # 调度器与任务管理
│       │   ├── syn_sched.c/h      # 协作式优先级调度器
│       │   ├── syn_task.h         # 任务描述符（~28 bytes）
│       │   ├── syn_timer.c/h      # 软件定时器
│       │   ├── syn_mailbox.h      # 无锁消息邮箱
│       │   ├── syn_watchdog.c/h   # 任务看门狗
│       │   └── syn_workqueue.c/h  # 工作队列
│       ├── control/               # 控制算法
│       │   ├── syn_pid.c/h        # PID 控制器（整数运算）
│       │   └── syn_autotune.c/h   # PID 自动调参
│       ├── motor/                 # 电机控制
│       │   ├── syn_foc.c/h        # FOC 变换（Clarke/Park/SVPWM）
│       │   ├── syn_motor_ctrl.c/h # 电机控制器
│       │   ├── syn_stepper.c/h    # 步进电机
│       │   └── syn_servo.c/h      # 舵机控制
│       ├── dsp/                   # 数字信号处理
│       ├── net/                   # 网络协议栈
│       │   ├── syn_mqtt.c/h       # MQTT 客户端
│       │   ├── syn_http.c/h       # HTTP 客户端
│       │   ├── syn_httpd.c/h      # HTTP 服务器
│       │   ├── syn_websocket.c/h  # WebSocket 服务器
│       │   ├── syn_dns.c/h        # DNS 解析器
│       │   └── syn_coap.c/h       # CoAP 协议
│       ├── crypto/                # 密码学模块
│       ├── cli/                   # 命令行接口
│       ├── drivers/               # 硬件驱动抽象
│       ├── display/               # 图形显示引擎
│       └── proto/                 # 通信协议
├── port/                          # 平台移植代码
├── examples/                      # 示例项目（20+）
├── docs/                          # MkDocs 文档
└── CMakeLists.txt                 # CMake 构建配置
```

### 模块职责表

| 模块目录 | 职责 | 关键文件 |
|----------|------|----------|
| `pt/` | 无栈协程核心，Duff's device 实现 | `syn_pt.h` |
| `sched/` | 优先级调度、定时器、邮箱、看门狗 | `syn_sched.c` |
| `control/` | PID 控制器、自动调参 | `syn_pid.c` |
| `motor/` | FOC 变换、电机控制、步进/舵机 | `syn_foc.c` |
| `dsp/` | FFT、IIR 滤波、卡尔曼滤波 | `syn_filter.c` |
| `net/` | MQTT/HTTP/WebSocket/DNS/CoAP | `syn_mqtt.c` |
| `crypto/` | BLAKE2s、ChaCha20-Poly1305、X25519 | `syn_blake2s.c` |
| `cli/` | 串口命令行、历史记录、自动补全 | `syn_cli.c` |
| `drivers/` | GPIO/UART/I2C/SPI/ADC/DMA/CAN | `syn_gpio.c` |

---

## 核心代码深度分析

### 1. Protothread 协程引擎（`syn_pt.h`）

SyntropicOS 的核心是 protothread 协程机制。它使用 C 语言的 `switch`/`__LINE__` 技巧（Duff's device）实现无栈协程，每个协程仅消耗 2 字节 RAM。

#### 1.1 协程控制块与状态定义

```c
typedef struct {
    uint16_t lc;   /**< Line continuation — stores __LINE__ at last yield */
} SYN_PT;
```

> **解析**：`SYN_PT` 结构体只有一个 `uint16_t lc` 成员，用于存储上次 yield 时的代码行号。这就是"2 字节 RAM"的来源——整个协程的执行状态压缩在一个 16 位整数中。

```c
typedef enum {
    PT_WAITING = 0,  /**< Thread is blocked (condition not yet met)  */
    PT_YIELDED = 1,  /**< Thread voluntarily yielded — call again    */
    PT_EXITED  = 2,  /**< Thread ran to PT_END — normal completion   */
    PT_ENDED   = 3,  /**< Thread was explicitly ended via PT_EXIT    */
} SYN_PT_Status;
```

> **解析**：四种返回状态精确描述协程的执行情况。`PT_WAITING` 表示条件未满足（如延时未到），`PT_YIELDED` 表示主动让出 CPU，`PT_EXITED` 表示正常结束，`PT_ENDED` 表示通过 `PT_EXIT` 提前终止。调度器根据返回值决定后续调度策略。

#### 1.2 PT_BEGIN / PT_END —— 协程入口与出口

```c
#define PT_BEGIN(pt)                                          \
    do {                                                       \
        char _pt_yield_flag = 1;                               \
        (void)_pt_yield_flag;                                  \
        switch ((pt)->lc) {                                    \
        case 0:
```

> **逐行解析**：
> - `do { ... } while (0)`：标准 C 宏安全包装，确保宏展开后表现为单条语句。
> - `char _pt_yield_flag = 1`：yield 标志位，初始为 1 表示"尚未 yield"。`PT_YIELD` 会将其设为 0 后返回，下次进入时检查该标志决定是继续执行还是再次 yield。
> - `switch ((pt)->lc)`：这是 Duff's device 的核心。`lc` 存储了上次 yield 时的 `__LINE__` 值。当协程首次执行时 `lc = 0`，匹配 `case 0:` 从头开始。当协程从 yield 恢复时，`switch` 直接跳转到对应的 `case __LINE__:` 标签，实现"断点续传"。

```c
#define PT_END(pt)                                            \
        }                                                      \
        _pt_yield_flag = 0;                                    \
        PT_INIT(pt);                                           \
        return PT_EXITED;                                      \
    } while (0)
```

> **解析**：`PT_END` 关闭 `switch` 块，重置 `lc` 为 0（通过 `PT_INIT`），返回 `PT_EXITED`。重置后协程可以在下次调用时从头开始执行，支持循环重启场景。

#### 1.3 PT_WAIT_UNTIL —— 条件阻塞

```c
#define PT_WAIT_UNTIL(pt, cond)                               \
    do {                                                       \
        (pt)->lc = __LINE__;                                   \
        SYN_FALLTHROUGH; /* intentional Duff's device */      \
        case __LINE__:                                         \
        if (!(cond)) {                                         \
            return PT_WAITING;                                 \
        }                                                      \
    } while (0)
```

> **深度解析**：这是 protothread 机制最精妙的宏。展开后的执行流程：
> 1. `(pt)->lc = __LINE__`：将当前行号保存到 `lc`。这是"存档点"。
> 2. `SYN_FALLTHROUGH`：显式 fall-through 注解（C23/C2x），告诉编译器这里是有意的 case 贯穿，不是遗漏 break。
> 3. `case __LINE__:`：与上面的 `__LINE__` 相同的行号作为 case 标签。当协程被再次调用时，`switch((pt)->lc)` 会跳转到这里。
> 4. `if (!(cond))`：检查条件是否满足。不满足则返回 `PT_WAITING`，调度器跳过该任务，下次再来检查。满足则继续往下执行。
>
> 整个过程没有任何栈保存/恢复操作——执行位置完全由 `lc` 中的行号决定。

#### 1.4 PT_TASK_DELAY_MS —— 非阻塞延时

```c
#define PT_DELAY_MS(pt, target, ms)                                             \
    do {                                                                        \
        *(target) = syn_port_get_tick_ms() + (uint32_t)(ms);                   \
        PT_WAIT_UNTIL(pt, (int32_t)(syn_port_get_tick_ms() - *(target)) >= 0); \
    } while (0)

#define PT_TASK_DELAY_MS(pt, task, ms) \
    PT_DELAY_MS(pt, &(task)->delay_until, ms)
```

> **解析**：
> - 首先计算截止时刻 `target = 当前时间 + 延时毫秒数`，存储在任务的 `delay_until` 字段中。
> - 然后调用 `PT_WAIT_UNTIL` 等待条件 `(int32_t)(syn_port_get_tick_ms() - *(target)) >= 0`。
> - **关键点**：使用有符号减法 `(int32_t)` 处理 32 位时间戳的回绕问题。当 `tick` 从 `0xFFFFFFFF` 回绕到 `0` 时，有符号减法仍然能正确判断时间是否已过截止点。
> - 这是**非阻塞延时**——在等待期间，调度器会运行其他任务，CPU 不会空转。

#### 1.5 PT_BLOCK_EVENT —— 事件阻塞

```c
#define PT_BLOCK_EVENT(pt, task, grp, mask)                    \
    do {                                                       \
        (task)->wait_event = (struct SYN_EventGroup *)(grp);   \
        (task)->wait_mask  = (mask);                            \
        (task)->state = (uint8_t)SYN_TASK_BLOCKED;             \
        PT_YIELD(pt);                                          \
        (grp)->flags &= ~(mask);  /* Auto-clear matched */    \
    } while (0)
```

> **解析**：与 `PT_WAIT_UNTIL`（每次调度都轮询条件）不同，`PT_BLOCK_EVENT` 将任务状态设为 `SYN_TASK_BLOCKED`。调度器在扫描时会**完全跳过** BLOCKED 状态的任务，直到事件组的 `flags` 与 `mask` 匹配。这实现了真正的**阻塞式等待**而无需抢占式上下文切换——既节省 CPU 周期，又支持 tickless 深度睡眠。事件触发后自动清除匹配的标志位，避免重复唤醒。

#### 1.6 PT_SPAWN —— 子协程

```c
#define PT_SPAWN(pt, child, func)                             \
    do {                                                       \
        PT_INIT(child);                                        \
        (pt)->lc = __LINE__;                                   \
        SYN_FALLTHROUGH;                                      \
        case __LINE__:                                         \
        if ((func) < PT_EXITED) {                              \
            return PT_WAITING;                                 \
        }                                                      \
    } while (0)
```

> **解析**：`PT_SPAWN` 允许在一个协程中启动子协程并等待其完成。首先初始化子协程（`PT_INIT`），然后反复调用子协程函数。如果子协程返回值 `< PT_EXITED`（即 `PT_WAITING` 或 `PT_YIELDED`），父协程返回 `PT_WAITING` 让出 CPU。子协程完成后（返回 `PT_EXITED` 或 `PT_ENDED`），父协程继续执行。这实现了协程的层次化组合。

---

### 2. 协作式调度器（`syn_sched.c`）

调度器是 SyntropicOS 的"心脏"，负责管理所有 protothread 任务的调度。

#### 2.1 调度器数据结构

```c
typedef struct SYN_Sched {
    SYN_Task  *tasks;         /**< Pointer to caller-owned task array    */
    size_t     task_count;    /**< Number of tasks in the array          */
    size_t     rr_per_prio[SYN_SCHED_PRIO_LEVELS];
                              /**< Per-priority round-robin indices      */
} SYN_Sched;
```

> **解析**：调度器结构体极其简洁。`tasks` 指向**调用者拥有**的任务数组——调度器不分配任何内存。`rr_per_prio` 是每个优先级的轮转索引，用于实现同优先级任务的 round-robin 调度。`SYN_SCHED_PRIO_LEVELS` 默认为 8 级。

#### 2.2 任务描述符

```c
typedef struct SYN_Task {
    SYN_PT          pt;           /**< Protothread continuation (2 bytes)      */
    SYN_TaskFunc    func;         /**< The task's protothread function          */
    const char      *name;         /**< Human-readable name (for debug/logging)  */
    uint8_t          priority;     /**< 0 = highest priority                     */
    uint8_t          state;        /**< SYN_TaskState                           */
    uint32_t         delay_until;  /**< Tick deadline for PT_TASK_DELAY_MS       */
    void            *user_data;    /**< Optional pointer to task-private state   */
    struct SYN_EventGroup *wait_event;  /**< Event group task blocks on (NULL if not blocking) */
    uint32_t         wait_mask;    /**< Bitmask of event flags to wait for       */
} SYN_Task;
```

> **解析**：任务描述符约 28 字节（32 位平台）。包含：
> - `pt`：protothread 控制块（2 字节）
> - `func`：任务函数指针
> - `priority`：优先级（0 最高）
> - `state`：任务状态（READY/SUSPENDED/DEAD/DEFERRED/BLOCKED/WAITING）
> - `delay_until`：延时截止时刻
> - `user_data`：用户私有数据指针
> - `wait_event` / `wait_mask`：事件阻塞信息
>
> 对比 FreeRTOS 的 TCB（通常 80~200+ 字节加 1KB+ 栈），SyntropicOS 的内存效率极高。

#### 2.3 任务状态机

```c
typedef enum {
    SYN_TASK_READY     = 0,  /**< Eligible to run on next scheduler tick */
    SYN_TASK_SUSPENDED = 1,  /**< Paused — skipped by scheduler          */
    SYN_TASK_DEAD      = 2,  /**< Exited — will not run again            */
    SYN_TASK_DEFERRED  = 3,  /**< Deferred — skipped for one pass        */
    SYN_TASK_BLOCKED   = 4,  /**< Blocked on event — skipped until fired */
    SYN_TASK_WAITING   = 5,  /**< PT_WAIT condition false — skip this tick */
} SYN_TaskState;
```

> **解析**：六种状态构成完整生命周期。特别注意 `DEFERRED` 和 `WAITING` 的区别：
> - `WAITING` 是 tick 局部状态——当 `PT_WAIT_UNTIL` 条件为假时设置，当前 tick 结束后自动清除。
> - `DEFERRED` 跨 tick 持续——通过 `PT_DEFER` 设置，任务在下一 tick 被跳过，实现"让位一次"语义。

#### 2.4 调度核心：`syn_sched_run()`

```c
bool syn_sched_run(SYN_Sched *sched)
{
    SYN_ASSERT(sched != NULL);
    const size_t n = sched->task_count;
    if (n == 0) {
        return false;
    }
    uint32_t now = syn_port_get_tick_ms();
    bool any_alive = false;
    SYN_METRIC_INC(sched_ticks);
```

> **解析**：每次调用 `syn_sched_run()` 执行一个调度 tick。首先获取当前时间戳 `now`，用于后续的延时检查。`sched_ticks` 是性能计数器。

```c
    SYN_Task *executed_task = NULL;
    size_t    executed_idx  = 0;
    size_t    attempts      = 0;
    while (attempts < n) {
        SYN_Task *best_task = NULL;
        size_t    best_idx  = 0;
        uint8_t   best_prio = 255;
        size_t    best_dist = n;   /* Larger than any valid distance */
```

> **解析**：外层 `while` 循环实现"等待重试"机制。当一个高优先级任务返回 `PT_WAITING`（条件未满足），调度器不会直接结束这个 tick，而是将其标记为 `WAITING` 并立即重新扫描寻找下一个可执行任务。`attempts` 限制重试次数不超过任务总数，防止无限循环。

```c
        for (size_t i = 0; i < n; i++) {
            SYN_Task *task = &sched->tasks[i];
            if (task->state == (uint8_t)SYN_TASK_DEAD) {
                continue;
            }
            any_alive = true;
            if (task->state == (uint8_t)SYN_TASK_SUSPENDED ||
                task->state == (uint8_t)SYN_TASK_DEFERRED  ||
                task->state == (uint8_t)SYN_TASK_WAITING) {
                continue;
            }
```

> **解析**：内层 `for` 遍历所有任务。先跳过 `DEAD` 任务（已终止）。然后跳过 `SUSPENDED`（被挂起）、`DEFERRED`（主动让位）、`WAITING`（条件未满足）状态的任务。这三种状态都不参与本轮调度。

```c
            /* Blocked on event — check if the event has fired */
            if (task->state == (uint8_t)SYN_TASK_BLOCKED) {
                if (task->wait_event != NULL &&
                    (task->wait_event->flags & task->wait_mask)) {
                    /* Event fired — transition to READY */
                    task->wait_event = NULL;
                    task->state = (uint8_t)SYN_TASK_READY;
                    /* Fall through to normal priority evaluation */
                } else {
                    continue;  /* Still blocked */
                }
            }
```

> **解析**：对于 `BLOCKED` 状态的任务，检查其等待的事件是否已触发。事件触发则转为 `READY` 状态并继续参与优先级评估；未触发则跳过。这是一种**惰性唤醒**——不需要中断回调去修改任务状态，调度器在扫描时自然发现事件已触发。

```c
            /* Delay check — signed arithmetic for wraparound safety */
            if (task->delay_until != 0 && (int32_t)(now - task->delay_until) < 0) {
                continue;
            }
```

> **解析**：延时检查使用有符号算术 `(int32_t)(now - task->delay_until) < 0`。当 `now` 从 `0xFFFFFFFF` 回绕到 `0` 时，`now - delay_until` 的无符号结果是一个巨大的正数，但有符号转换后变为负数，正确表示"时间未到"。这是嵌入式系统中处理 32 位时间戳回绕的标准技巧。

```c
            /* Task is ready — compute rotation distance for its priority */
            const uint8_t prio = task->priority;
            size_t rr_start = sched->rr_per_prio[prio];
            if (rr_start >= n) { rr_start = 0; } /* Defensive clamp */
            const size_t dist = (i >= rr_start)
                              ? (i - rr_start)
                              : (n - rr_start + i);
            if (prio < best_prio ||
                (prio == best_prio && dist < best_dist)) {
                best_prio = prio;
                best_dist = dist;
                best_task = task;
                best_idx  = i;
            }
```

> **解析**：这段实现了**优先级 + round-robin** 调度策略：
> 1. 优先级数值越小优先级越高（`prio < best_prio` 时更新最优）。
> 2. 同优先级时，计算当前任务相对于轮转起点的"距离" `dist`，距离最小的优先执行。
> 3. 距离计算考虑了数组回绕：`i >= rr_start` 时距离为 `i - rr_start`，否则为 `n - rr_start + i`（绕回数组头部）。
> 4. `rr_start` 防御性钳位到 `[0, n)` 范围内。

```c
        if (best_task == NULL) {
            break;  /* No eligible tasks this tick */
        }
        SYN_PT_Status status = sched_run_task(best_task);
        if (status == PT_WAITING) {
            best_task->state = (uint8_t)SYN_TASK_WAITING;
            attempts++;
            continue;
        }
```

> **解析**：如果找到可执行任务，调用其 protothread 函数。如果返回 `PT_WAITING`，将任务标记为 `WAITING` 并递增 `attempts`，继续 `while` 循环寻找下一个任务。这就是"等待重试"机制——高优先级任务条件未满足时，低优先级任务仍有机会在本 tick 执行。

```c
        executed_task = best_task;
        executed_idx  = best_idx;
        SYN_METRIC_INC(sched_switches);
        if (best_task->state != (uint8_t)SYN_TASK_DEFERRED) {
            const size_t next = executed_idx + 1;
            sched->rr_per_prio[best_prio] = (next >= n) ? 0 : next;
        }
        break;  /* One useful execution per tick */
    }
```

> **解析**：任务执行了实际工作后，更新 round-robin 指针到下一个任务（除非任务通过 `PT_DEFER` 自行延期）。注意**每个 tick 只执行一个有用任务**——这是协作式调度的核心特征，确保每个任务获得完整的执行窗口，不会被抢占。

#### 2.5 Tickless 低功耗模式

```c
SYN_NORETURN void syn_sched_run_tickless_ex(SYN_Sched *sched,
                                             SYN_Sleep *sleep,
                                             SYN_Timer *timers,
                                             size_t timer_count)
{
    for (;;) {
        syn_sched_run(sched);
        syn_timer_service(timers, timer_count);
        uint32_t now = syn_port_get_tick_ms();
        uint32_t task_wake  = syn_sched_next_wakeup(sched);
        uint32_t timer_wake = syn_timer_next_expiry(timers, timer_count);
        uint32_t wake = task_wake;
        if (timer_wake != UINT32_MAX &&
            (wake == UINT32_MAX || (int32_t)(timer_wake - wake) < 0)) {
            wake = timer_wake;
        }
        if (wake != now && !syn_sleep_any_locked(sleep)) {
            if (wake == UINT32_MAX) {
                syn_sleep_enter(sleep);
            } else {
                syn_port_sleep_until(wake);
            }
        }
    }
}
```

> **逐段解析**：
> 1. 每轮先运行调度器 `syn_sched_run()`，然后服务软件定时器。
> 2. 计算两个唤醒源：任务的最早截止时间 `task_wake` 和定时器的最早过期时间 `timer_wake`。
> 3. 取两者中较早的作为唤醒时刻 `wake`。
> 4. 如果没有任务立即可执行（`wake != now`）且没有 sleep lock 持有，则进入低功耗：
>    - `wake == UINT32_MAX`：无任何未来截止时间，进入轻睡眠等待中断唤醒。
>    - 否则：调用 `syn_port_sleep_until(wake)` 精确睡眠到指定时刻。
>
> 这种设计使 SyntropicOS 在电池供电设备上实现极低待机功耗——CPU 在无任务可执行时自动进入深度睡眠。

---

### 3. PID 控制器（`syn_pid.c`）

SyntropicOS 的 PID 控制器完全使用**整数运算**，无浮点依赖，适合无 FPU 的微控制器。

#### 3.1 初始化与抗积分饱和配置

```c
void syn_pid_init(SYN_PID *pid, const SYN_PID_Config *cfg)
{
    SYN_ASSERT(pid != NULL);
    SYN_ASSERT(cfg != NULL);
    SYN_ASSERT(cfg->scale != 0);
    memset(pid, 0, sizeof(*pid));
    pid->cfg   = *cfg;
    pid->first = true;
    if (pid->cfg.integral_max == 0) {
        if (pid->cfg.ki > 0) {
            pid->cfg.integral_max = (int32_t)(
                ((int64_t)pid->cfg.out_max * pid->cfg.scale * 1000)
                / pid->cfg.ki);
        } else {
            pid->cfg.integral_max = pid->cfg.out_max * pid->cfg.scale;
        }
    }
}
```

> **解析**：
> - `memset(pid, 0, sizeof(*pid))`：清零所有状态（积分项、上次误差等）。
> - `pid->first = true`：标记首次调用，避免首次微分项计算出现巨大跳变。
> - **自动计算积分上限**：如果用户未设置 `integral_max`，根据输出上限 `out_max`、缩放因子 `scale` 和积分增益 `ki` 自动推导。公式为 `integral_max = out_max * scale * 1000 / ki`。使用 `int64_t` 中间变量防止 32 位乘法溢出。
> - `scale` 是定点缩放因子，用于将浮点 PID 参数转换为整数运算。例如 `scale = 100` 表示 Kp=1.5 对应 `kp = 150`。

#### 3.2 PID 更新核心算法

```c
int32_t syn_pid_update(SYN_PID *pid, int32_t setpoint,
                        int32_t measured, uint32_t dt_ms)
{
    SYN_ASSERT(pid != NULL);
    if (dt_ms == 0) dt_ms = 1;
    int32_t error = setpoint - measured;
```

> **解析**：`dt_ms == 0` 防护避免除零错误。误差 `error = 设定值 - 测量值`，正误差表示需要增加输出。

```c
    /* ── Proportional ──────────────────────────────────────────────────── */
    int32_t p_term = (pid->cfg.kp * error) / pid->cfg.scale;
```

> **解析**：比例项 `P = Kp * error / scale`。`scale` 将整数化的增益还原为实际值。例如 `kp = 150, scale = 100, error = 10` 时，`p_term = 150 * 10 / 100 = 15`。

```c
    /* ── Integral (with anti-windup) ───────────────────────────────────── */
    pid->integral += error * (int32_t)dt_ms;
    if (pid->cfg.integral_max > 0) {
        pid->integral = SYN_CLAMP(pid->integral,
                              -pid->cfg.integral_max,
                               pid->cfg.integral_max);
    }
    int32_t i_term = ((pid->cfg.ki * pid->integral) / 1000) / pid->cfg.scale;
```

> **解析**：
> - 积分项累加 `error * dt_ms`（误差对时间的积分）。
> - `SYN_CLAMP` 将积分项限制在 `[-integral_max, +integral_max]` 范围内，防止**积分饱和（windup）**——当输出长时间饱和时，积分项会持续增长导致恢复时严重超调。
> - **两步除法技巧**：`((ki * integral) / 1000) / scale` 而非 `(ki * integral) / (scale * 1000)`。先除以 1000（时间归一化）再除以 scale，避免分母过大导致整数截断误差。注释中明确说明了这一改进。

```c
    /* ── Derivative ────────────────────────────────────────────────────── */
    int32_t d_raw;
    if (pid->first) {
        d_raw = 0;
        pid->first = false;
    } else {
        d_raw = ((error - pid->prev_error) * 1000) / (int32_t)dt_ms;
    }
    pid->prev_error = error;
```

> **解析**：
> - 首次调用时 `d_raw = 0`，避免因 `prev_error` 为零导致微分项巨大跳变。
> - 微分项 `D = (error - prev_error) * 1000 / dt_ms`，即误差变化率。乘以 1000 是因为 `dt_ms` 单位是毫秒，而标准 PID 公式中 dt 单位是秒。

```c
    /* Optional EMA filter on derivative */
    if (pid->cfg.d_filter_alpha > 0 && pid->cfg.d_filter_alpha < 255) {
        int32_t alpha = pid->cfg.d_filter_alpha;
        pid->prev_d_filtered += (alpha * (d_raw - pid->prev_d_filtered)) >> 8;
        d_raw = pid->prev_d_filtered;
    }
    int32_t d_term = (pid->cfg.kd * d_raw) / pid->cfg.scale;
```

> **解析**：
> - **EMA（指数移动平均）滤波器**：可选的微分项滤波。`alpha` 是 0~255 的 8 位定点系数（相当于 0~1 的小数）。公式 `filtered += alpha * (raw - filtered) / 256` 是标准的一阶 IIR 滤波器，用位移代替除法提高效率。
> - 滤波后 `d_term = Kd * d_filtered / scale`。
> - 微分项滤波是实际控制中的常见需求——测量噪声会被微分放大，EMA 滤波有效抑制高频噪声。

```c
    /* ── Sum and clamp ─────────────────────────────────────────────────── */
    int32_t output = p_term + i_term + d_term;
    output = SYN_CLAMP(output, pid->cfg.out_min, pid->cfg.out_max);
    /* Anti-windup: if output is saturated, freeze integral */
    if ((output == pid->cfg.out_max && error > 0) ||
        (output == pid->cfg.out_min && error < 0)) {
        pid->integral -= error * (int32_t)dt_ms;
    }
    pid->output = output;
    return output;
}
```

> **解析**：
> - 三项求和后钳位到 `[out_min, out_max]`。
> - **条件积分冻结（conditional integration anti-windup）**：当输出已饱和且误差方向与饱和方向一致时（输出达上限且误差为正，或输出达下限且误差为负），**回滚**刚才的积分累加 `integral -= error * dt_ms`。这比简单的积分限幅更智能——它只在"积分有害"时才冻结，避免了饱和恢复时的超调。

---

### 4. FOC 电机控制（`syn_foc.c`）

SyntropicOS 实现了完整的**磁场定向控制（FOC）**变换链，全部使用 Q16.16 定点运算。

#### 4.1 Clarke 变换（三相 → 两相静止）

```c
/** @brief Precomputed 1/√3 in Q16.16 (≈ 0.57735). */
#define Q16_INV_SQRT3     37837

void syn_foc_clarke(const SYN_FOC_ABC *abc, SYN_FOC_AB *ab)
{
    ab->alpha = abc->a;
    ab->beta = q16_mul(abc->a + 2 * abc->b, Q16_INV_SQRT3);
}
```

> **解析**：
> - Clarke 变换将三相电流 (a, b, c) 转换为两相正交分量 (α, β)。
> - 等幅变换简化形式：`α = a`，`β = (a + 2b) / √3`。
> - `Q16_INV_SQRT3 = 37837` 是 1/√3 的 Q16.16 定点表示：`0.57735 * 65536 ≈ 37837`。
> - `q16_mul` 执行定点乘法：`(a * b) >> 16`，使用 `int64_t` 中间值防止溢出。
> - 这里假设 `a + b + c ≈ 0`（三相平衡），因此省略了 c 分量。

#### 4.2 Park 变换（静止 → 旋转坐标系）

```c
void syn_foc_park(const SYN_FOC_AB *ab, q16_t theta, SYN_FOC_DQ *dq)
{
    q16_t cos_t = q16_cos(theta);
    q16_t sin_t = q16_sin(theta);
    dq->d = q16_mul(ab->alpha, cos_t) + q16_mul(ab->beta, sin_t);
    dq->q = -q16_mul(ab->alpha, sin_t) + q16_mul(ab->beta, cos_t);
}
```

> **解析**：
> - Park 变换将静止 α-β 坐标系转换到与转子同步旋转的 d-q 坐标系。
> - `d = α·cos(θ) + β·sin(θ)`：直轴分量（磁通分量）。
> - `q = -α·sin(θ) + β·cos(θ)`：交轴分量（转矩分量）。
> - `q16_cos` 和 `q16_sin` 是定点三角函数，使用查表或泰勒展开实现。
> - 变换后 d-q 分量在稳态下为直流值，使 PI 控制器能实现零稳态误差。

#### 4.3 SVPWM 空间矢量调制

```c
void syn_foc_svpwm(const SYN_FOC_AB *ab, q16_t v_bus,
                   q16_t *duty_a, q16_t *duty_b, q16_t *duty_c)
{
    q16_t s3b = (q16_t)(((int64_t)ab->beta * 113512) >> 16);  /* √3·β */
    q16_t va = ab->alpha;
    q16_t vb = (-ab->alpha + s3b) / 2;
    q16_t vc = (-ab->alpha - s3b) / 2;
```

> **解析**：
> - SVPWM 的第一步是从 α-β 电压计算三相参考电压。
> - `113512` 是 √3 的 Q16.16 表示：`1.73205 * 65536 ≈ 113512`。
> - `va = α`，`vb = (-α + √3·β) / 2`，`vc = (-α - √3·β) / 2`。

```c
    q16_t v_min = va;
    q16_t v_max = va;
    if (vb < v_min) v_min = vb;
    if (vc < v_min) v_min = vc;
    if (vb > v_max) v_max = vb;
    if (vc > v_max) v_max = vc;
    q16_t v_offset = -(v_max + v_min) / 2;
```

> **解析**：**中心钳位（center-clamping）**技术。计算三相电压的最小值和最大值，然后计算偏移量 `v_offset = -(v_max + v_min) / 2`。这个偏移量将三相电压的中点对齐到母线电压的一半，最大化线性调制范围，同时实现零序电压注入。

```c
    *duty_a = q16_clamp(q16_div(va + v_offset, v_bus) + Q16_HALF, 0, Q16_ONE);
    *duty_b = q16_clamp(q16_div(vb + v_offset, v_bus) + Q16_HALF, 0, Q16_ONE);
    *duty_c = q16_clamp(q16_div(vc + v_offset, v_bus) + Q16_HALF, 0, Q16_ONE);
```

> **解析**：
> - 将电压归一化为 [0, 1] 范围的占空比：`duty = (v + offset) / v_bus + 0.5`。
> - `Q16_HALF` 和 `Q16_ONE` 是 0.5 和 1.0 的 Q16.16 表示。
> - `q16_clamp` 确保占空比在 [0, 1] 范围内，防止过调制。
> - 最终三相占空比直接送入 PWM 外设的比较寄存器。

---

### 5. 命令行接口（`syn_cli.c`）

SyntropicOS 内置了一个功能完善的串口 CLI，支持命令历史、ANSI 转义序列解析和内置命令。

#### 5.1 命令分词器

```c
static int cli_tokenize(char *line, char *argv[], int max_args)
{
    int argc = 0;
    char *p = line;
    while (*p && argc < max_args) {
        while (*p == ' ' || *p == '\t') p++;
        if (*p == '\0') break;
        if (*p == '"') {
            p++;
            argv[argc++] = p;
            while (*p && *p != '"') p++;
            if (*p == '"') *p++ = '\0';
        } else {
            argv[argc++] = p;
            while (*p && *p != ' ' && *p != '\t') p++;
            if (*p) *p++ = '\0';
        }
    }
    return argc;
}
```

> **逐段解析**：
> - 该函数**原地修改**输入缓冲区 `line`，将空格/Tab 分隔的 token 指针存入 `argv[]`。
> - 跳过前导空白字符。
> - **双引号字符串处理**：遇到 `"` 时，跳过开引号，记录 token 起始位置，扫描到闭引号并将其替换为 `\0`。这允许命令参数中包含空格，例如 `set name "hello world"`。
> - 普通 token：扫描到下一个空白字符，将其替换为 `\0` 终止当前 token。
> - 返回 token 数量 `argc`，与标准 C `main(argc, argv)` 模式一致。

#### 5.2 命令分发

```c
static void cli_dispatch(SYN_CLI *cli, char *line)
{
    char *argv[SYN_CLI_MAX_ARGS];
    int argc = cli_tokenize(line, argv, SYN_CLI_MAX_ARGS);
    if (argc == 0) return;

    if (strcmp(argv[0], "help") == 0 || strcmp(argv[0], "?") == 0) {
        cli_puts(cli, "Available commands:\r\n");
        cli_puts(cli, "  help           -- Show this help\r\n");
        // ... 列出所有命令 ...
        for (size_t i = 0; i < cli->command_count; i++) {
            cli_puts(cli, "  ");
            cli_puts(cli, cli->commands[i].name);
            // ... 对齐和帮助文本 ...
        }
        return;
    }
    // ... 其他内置命令检查 ...

    for (size_t i = 0; i < cli->command_count; i++) {
        if (strcmp(argv[0], cli->commands[i].name) == 0) {
            if (cli->commands[i].handler != NULL) {
                int ret = cli->commands[i].handler(argc, argv);
                if (ret != 0) {
                    // ... 格式化错误输出 ...
                }
            }
            return;
        }
    }
    cli_puts(cli, "Unknown command: ");
    cli_puts(cli, argv[0]);
    cli_puts(cli, "\r\nType 'help' for a list of commands.\r\n");
}
```

> **解析**：
> - 命令分发采用三级查找：先检查内置命令（help/version/uptime/errors/tasks），再遍历用户注册的命令表，最后输出"未知命令"提示。
> - 用户命令通过 `SYN_CLI_Command` 结构体注册，包含 `name`、`handler` 函数指针和 `help` 文本。
> - handler 返回非零值时自动格式化错误输出。
> - 内置 `help` 命令动态列出所有可用命令（包括用户注册的），并对齐显示。

#### 5.3 字符处理与 ANSI 转义

```c
void syn_cli_process_char(SYN_CLI *cli, char ch)
{
    /* ANSI escape sequence parsing (e.g. Up Arrow = \x1B[A) */
    if (cli->escape_state == 1) {
        if (ch == '[') cli->escape_state = 2;
        else cli->escape_state = 0;
        return;
    }
    if (cli->escape_state == 2) {
        cli->escape_state = 0;
        if (ch == 'A') ch = 0x10; /* Map Up arrow to Ctrl-P */
        else return;
    }
    if (ch == 0x1B) {
        cli->escape_state = 1;
        return;
    }
```

> **解析**：ANSI 转义序列解析使用两状态机：
> - 状态 1：收到 `0x1B`（ESC），等待 `[` 确认是 CSI 序列。
> - 状态 2：收到 `[`，等待最终字符。`A` 表示上箭头，映射为 `Ctrl-P`（`0x10`）用于历史回溯。其他箭头键被忽略。
>
> 这种简化的 ANSI 解析足以支持终端的上箭头历史功能，同时保持极小的代码体积。

#### 5.4 命令历史

```c
static void cli_history_push(SYN_CLI *cli, const char *line)
{
    if (line[0] == '\0') return;
    if (cli->history_count > 0) {
        size_t last = (cli->history_write + SYN_CLI_HISTORY_DEPTH - 1)
                      % SYN_CLI_HISTORY_DEPTH;
        if (strcmp(cli->history[last], line) == 0) {
            cli->history_read = cli->history_write;
            return;
        }
    }
    strncpy(cli->history[cli->history_write], line, SYN_CLI_LINE_BUF_SIZE - 1);
    cli->history[cli->history_write][SYN_CLI_LINE_BUF_SIZE - 1] = '\0';
    cli->history_write = (cli->history_write + 1) % SYN_CLI_HISTORY_DEPTH;
    if (cli->history_count < SYN_CLI_HISTORY_DEPTH) {
        cli->history_count++;
    }
    cli->history_read = cli->history_write;
}
```

> **解析**：
> - 环形缓冲区实现命令历史，深度由 `SYN_CLI_HISTORY_DEPTH` 配置。
> - **去重**：如果新命令与最后一条历史相同，不重复存储，但重置读取游标。
> - `strncpy` 安全拷贝并强制终止，防止缓冲区溢出。
> - `history_write` 和 `history_read` 使用模运算实现环形索引。

---

### 6. ESP32 移植层（`port_esp32.c`）

SyntropicOS 通过移植层抽象硬件接口。以下是 ESP32 的移植实现：

```c
uint32_t syn_port_get_tick_ms(void)
{
    return (uint32_t)(esp_timer_get_time() / 1000);
}

void syn_port_enter_critical(void)
{
    portENTER_CRITICAL(&critical_mux);
}

void syn_port_exit_critical(void)
{
    portEXIT_CRITICAL(&critical_mux);
}
```

> **解析**：
> - `syn_port_get_tick_ms()` 包装 ESP-IDF 的 `esp_timer_get_time()`（微秒精度），返回毫秒时间戳。
> - 临界区使用 ESP32 的 spinlock 实现 `portENTER_CRITICAL` / `portEXIT_CRITICAL`，保护共享数据。

```c
SYN_Status syn_port_gpio_init(SYN_GPIO_Pin pin, SYN_GPIO_Mode mode)
{
    gpio_config_t io_conf = {};
    io_conf.pin_bit_mask = (1ULL << pin);
    io_conf.intr_type = GPIO_INTR_DISABLE;
    switch (mode) {
        case SYN_GPIO_INPUT:
            io_conf.mode = GPIO_MODE_INPUT;
            break;
        case SYN_GPIO_OUTPUT:
            io_conf.mode = GPIO_MODE_OUTPUT;
            break;
        case SYN_GPIO_INPUT_PULLUP:
            io_conf.mode = GPIO_MODE_INPUT;
            io_conf.pull_up_en = GPIO_PULLUP_ENABLE;
            break;
        // ... 其他模式 ...
    }
    esp_err_t err = gpio_config(&io_conf);
    return (err == ESP_OK) ? SYN_OK : SYN_INVALID_PARAM;
}
```

> **解析**：
> - GPIO 移植将 SyntropicOS 的抽象模式（INPUT/OUTPUT/PULLUP/PULLDOWN/OUTPUT_OD）映射到 ESP-IDF 的 `gpio_config_t` 配置。
> - `1ULL << pin` 使用 64 位掩码支持 ESP32 的所有 GPIO 引脚（最高 GPIO48）。
> - 返回统一的 `SYN_Status` 状态码，上层无需关心 ESP-IDF 的 `esp_err_t`。

```c
SYN_WEAK SYN_Status syn_port_uart_init(SYN_UARTInstance instance, uint32_t baudrate)
{
    uart_port_t port = get_uart_port(instance);
    uart_config_t uart_config = {
        .baud_rate = (int)baudrate,
        .data_bits = UART_DATA_8_BITS,
        .parity    = UART_PARITY_DISABLE,
        .stop_bits = UART_STOP_BITS_1,
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
        .source_clk = UART_SCLK_DEFAULT,
    };
    uart_param_config(port, &uart_config);
    uart_driver_install(port, 256, 256, 0, NULL, 0);
    uart_set_pin(port, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE,
                 UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);
    return SYN_OK;
}
```

> **解析**：
> - `SYN_WEAK` 关键字使该函数成为弱符号——用户可以在自己的代码中覆盖它，例如为 USB CDC 芯片（ESP32-S3/C3）提供不同的串口实现。
> - UART 配置为标准 8N1 模式，RX/TX 缓冲区各 256 字节。
> - `UART_PIN_NO_CHANGE` 使用默认引脚映射，用户可通过 `syn_port_uart_init` 覆盖。

---

## API 使用指南

### 最小示例：LED 闪烁

```c
#include "syntropic/syntropic.h"

#define LED_PIN  2  /* ESP32 onboard LED */

static SYN_PT_Status blink(SYN_PT *pt, SYN_Task *task) {
    PT_BEGIN(pt);
    for (;;) {
        syn_gpio_toggle(LED_PIN);
        PT_TASK_DELAY_MS(pt, task, 500);
    }
    PT_END(pt);
}

int main(void) {
    syn_gpio_init(LED_PIN, SYN_GPIO_OUTPUT);
    static SYN_Task tasks[1];
    static SYN_Sched sched;
    syn_task_create(&tasks[0], "blink", blink, 0, NULL);
    syn_sched_init(&sched, tasks, 1);
    syn_sched_run_forever(&sched);
}
```

### 多任务示例：PID 温控 + 串口 CLI

```c
#include "syntropic/syntropic.h"

static SYN_PID  pid;
static SYN_CLI  cli;
static SYN_Task tasks[3];

/* PID 温控任务 */
static SYN_PT_Status temp_control(SYN_PT *pt, SYN_Task *task) {
    PT_BEGIN(pt);
    SYN_PID_Config cfg = {
        .kp = 200, .ki = 50, .kd = 10, .scale = 100,
        .out_min = 0, .out_max = 4095
    };
    syn_pid_init(&pid, &cfg);
    
    for (;;) {
        int32_t measured = syn_adc_read(TEMP_SENSOR_CH);
        int32_t output = syn_pid_update(&pid, 5500, measured, 100);
        syn_dac_write(HEATER_CH, output);
        PT_TASK_DELAY_MS(pt, task, 100);  /* 10Hz 采样 */
    }
    PT_END(pt);
}

/* 串口 CLI 任务 */
static SYN_PT_Status cli_task(SYN_PT *pt, SYN_Task *task) {
    PT_BEGIN(pt);
    static uint8_t buf[1];
    for (;;) {
        size_t n;
        syn_port_uart_receive_byte(0, buf, 10);
        if (n > 0) {
            syn_cli_process_char(&cli, (char)buf[0]);
        }
        PT_YIELD(pt);
    }
    PT_END(pt);
}

int main(void) {
    syn_port_serial_init(115200);
    
    static const SYN_CLI_Command commands[] = {
        {"setpoint", cmd_setpoint, "Set temperature setpoint"},
        {"tune",     cmd_tune,     "Tune PID parameters"},
    };
    syn_cli_init(&cli, commands, 2, "temp> ");
    
    syn_task_create(&tasks[0], "pid",  temp_control, 0, NULL);  /* 高优先级 */
    syn_task_create(&tasks[1], "cli",  cli_task,     1, NULL);  /* 低优先级 */
    syn_task_create(&tasks[2], "log",  log_task,     2, NULL);
    
    static SYN_Sched sched;
    syn_sched_init(&sched, tasks, 3);
    syn_sched_run_forever(&sched);
}
```

### cURL 测试：MQTT 数据上报

```bash
# 启动 MQTT broker
mosquitto -p 1883 &

# 订阅设备上报的主题
mosquitto_sub -h localhost -t "syntropic/sensor/temp" -v

# 模拟设备上报（如果设备通过 HTTP API 上报）
curl -X POST http://device-ip/api/telemetry \
  -H "Content-Type: application/json" \
  -d '{"temperature": 25.5, "humidity": 60.2, "pid_output": 2048}'
```

### Python 示例：串口通信

```python
import serial
import time

ser = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)

# 发送命令
def send_cmd(cmd):
    ser.write((cmd + '\r\n').encode())
    time.sleep(0.1)
    return ser.read_all().decode()

# 查看版本
print(send_cmd('version'))
# 输出示例: SyntropicOS v2026.7.1 (2026-07-24 10:30:00) git:abc1234

# 查看任务列表
print(send_cmd('tasks'))
# 输出示例:
# Tasks:
#   pid    prio=0  READY
#   cli    prio=1  READY
#   log    prio=2  READY

# 设置 PID 参数
print(send_cmd('tune kp=200 ki=50 kd=10'))
```

---

## 编译与部署

### 环境搭建

| 工具 | 版本要求 | 用途 |
|------|----------|------|
| CMake | ≥ 3.16 | 构建系统 |
| GCC ARM | ≥ 10.0 | STM32 交叉编译 |
| ESP-IDF | ≥ 5.0 | ESP32 开发框架 |
| Pico SDK | ≥ 1.5 | RP2040 开发 |
| Arduino IDE | ≥ 2.0 | Arduino 平台 |

### 方式一：CMake 集成（STM32 / RP2040）

```bash
# 1. 添加为 Git 子模块
cd your_project
git submodule add https://github.com/outlookhazy/SyntropicOS lib/SyntropicOS
git submodule update --init

# 2. 创建配置文件
cp lib/SyntropicOS/src/syntropic/syn_config_template.h include/syn_config.h

# 3. 在 CMakeLists.txt 中集成
cat >> CMakeLists.txt << 'EOF'
add_subdirectory(lib/SyntropicOS)
target_link_libraries(your_target PRIVATE syntropic)
EOF

# 4. 编译
mkdir build && cd build
cmake .. -DCMAKE_TOOLCHAIN_FILE=arm-none-eabi.cmake
make -j$(nproc)
```

#### 编译配置表

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `SYN_USE_SCHED` | 1 | 启用调度器 |
| `SYN_USE_PT` | 1 | 启用 protothread |
| `SYN_USE_PID` | 1 | 启用 PID 控制器 |
| `SYN_USE_FOC` | 1 | 启用 FOC 电机控制 |
| `SYN_USE_CLI` | 1 | 启用命令行接口 |
| `SYN_USE_MQTT` | 1 | 启用 MQTT 客户端 |
| `SYN_USE_TICKLESS` | 0 | 启用 tickless 低功耗 |
| `SYN_SCHED_PRIO_LEVELS` | 8 | 优先级级数 |
| `SYN_CLI_LINE_BUF_SIZE` | 128 | CLI 行缓冲区大小 |
| `SYN_LOG_LEVEL` | 1 (DEBUG) | 最低日志级别 |

### 方式二：ESP32 ESP-IDF

```bash
# 1. 创建 ESP-IDF 项目
idf.py create-project syntropic_demo
cd syntropic_demo

# 2. 添加 SyntropicOS 组件
cd components
git clone https://github.com/outlookhazy/SyntropicOS

# 3. 创建 CMakeLists.txt
cat > CMakeLists.txt << 'EOF'
idf_component_register(
    SRCS "main/main.c"
    INCLUDE_DIRS "main"
    REQUIRES SyntropicOS
)
EOF

# 4. 配置目标芯片
idf.py set-target esp32s3

# 5. 编译与烧录
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

### 方式三：Arduino IDE

```
1. 打开 Arduino IDE
2. 工具 → 管理库 → 搜索 "SyntropicOS" → 安装
3. 文件 → 示例 → SyntropicOS → Blink
4. 选择开发板 → 上传
```

### 烧录步骤

```bash
# STM32 (使用 OpenOCD + ST-Link)
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg \
  -c "program build/firmware.elf verify reset exit"

# ESP32 (使用 esptool)
esptool.py --port /dev/ttyUSB0 --baud 921600 \
  write_flash 0x10000 build/firmware.bin

# RP2040 (使用 OpenOCD)
openocd -f interface/cmsis-dap.cfg -f target/rp2040.cfg \
  -c "program build/firmware.elf verify reset exit"
```

---

## 项目亮点与适用场景

### 项目亮点

1. **极致内存效率**：每个任务仅 2~28 字节 RAM，整个 RTOS + 网络栈 + DSP 流水线可在 8KB RAM 的 AVR 上运行。
2. **协作式设计哲学**：所有 70+ 模块原生构建为非阻塞状态机，从根本上消除了"阻塞库 + 协作调度器"的阻抗失配问题。
3. **纯整数运算**：完整的 Q16.16 定点数学库（三角函数、矩阵、FFT、卡尔曼滤波），无需 FPU 和 `libm.a`。
4. **工业级协议支持**：SAE J1939、CANopen、Modbus RTU/TCP、NMEA 0183，适用于工程机械和船舶电子。
5. **安全性**：内置 ChaCha20-Poly1305 加密、X25519 密钥交换、BLAKE2s 哈希，支持 WireGuard VPN（`syn_wg.c`，29KB 实现）。
6. **可配置性**：每个模块通过 `#define` 独立开关，"用多少付多少"——未启用的模块完全不参与编译。

### 适用场景

| 场景 | 推荐平台 | 关键模块 |
|------|----------|----------|
| 电机驱动器（BLDC/PMSM） | STM32F4 | FOC + PID + SVPWM + CAN |
| IoT 传感器节点 | ESP32 | MQTT + HTTP + SNTP + WiFi |
| 电池管理系统（BMS） | STM32 | J1939 + ADC + PID |
| 工业网关 | RP2040 双核 | Modbus + MQTT + 串口多路 |
| 智能家居控制器 | Arduino | CLI + GPIO + 传感器驱动 |
| 低功耗数据记录器 | STM32L4 | Tickless + SD 卡 + RTC |
| 机器人运动控制 | STM32F4 | 插补器 + FOC + 编码器 |

---

## 总结

SyntropicOS 是一个设计哲学极其清晰的嵌入式框架——它不试图成为又一个 FreeRTOS 或 Zephyr，而是选择了**协作式 + 零堆分配 + 纯整数运算**这条差异化路线。通过 Duff's device 实现的 2 字节 protothread、70+ 原生非阻塞模块、完整的 FOC/DSP/网络协议栈，它在资源受限的微控制器上提供了通常只有大型 RTOS 才有的功能集。

项目创建于 2026 年 6 月 29 日，目前处于活跃开发阶段。对于需要在 8KB~64KB RAM 设备上实现复杂多任务逻辑、电机控制或 IoT 通信的开发者，SyntropicOS 是一个值得深入研究的方案。

---

📝 作者：蔡浩宇（jun-chy）  
📅 日期：2026-07-24  
🔗 项目地址：[https://github.com/outlookhazy/SyntropicOS](https://github.com/outlookhazy/SyntropicOS)
