# µCNC 通用数控固件深度解析 — 跨平台 CNC/G-Code 运动控制内核全链路代码分析

> **日期：** 2026-07-14  
> **分类：** 多平台 / 嵌入式 CNC  
> **Stars：** 445  
> **作者：** 蔡浩宇（jun-chy）

---

## 项目链接

| 属性 | 信息 |
|------|------|
| GitHub 地址 | [Paciente8159/uCNC](https://github.com/Paciente8159/uCNC) |
| 作者 | João Martins（Paciente8159） |
| 许可证 | GPL-3.0 |
| 最近更新 | 2026-07-14 |
| 主要语言 | C（GNU99） |
| 目标平台 | AVR / ESP32 / ESP8266 / STM32F0/F1/F4/H7 / RP2040 / RP2350 / SAMD21 / LPC176x |
| 当前版本 | v1.16 |
| 开发框架 | 裸机 / Arduino IDE / PlatformIO / Makefile |

---

## 项目简介

µCNC 是一款受 [Grbl](https://github.com/gnea/grbl) 和 [LinuxCNC](http://linuxcnc.org/) 启发的**通用数控（CNC）固件**，始于 2019 年下半年。它的设计目标是既有 Grbl 的紧凑与高效，又有 LinuxCNC 的灵活与模块化——通过标准化的 HAL（硬件抽象层）接口，将 CNC 运动控制代码与底层 MCU 架构完全解耦，只需实现必要的硬件接口即可移植到任意微控制器。

µCNC 支持 6 轴运动控制，兼容 Grbl 协议，可直接对接 Universal Gcode Sender（UGS）、CNCjs 等 GUI 上位机。其模块化架构允许用户通过配置文件启用/禁用功能模块，并通过事件钩子（Event Hooks）扩展核心行为。

### 核心特性一览

| # | 特性 | 说明 |
|:-:|------|------|
| 1 | 多平台 HAL | AVR/ESP32/STM32/RP2040/SAMD21/LPC176x 统一接口，移植仅需实现 MCU 层 |
| 2 | 6 轴运动控制 | 支持 Cartesian、CoreXY、Delta、Linear Delta、SCARA、R-Theta 等运动学 |
| 3 | Grbl 协议兼容 | 兼容 Grbl v1.1 协议，无缝对接现有 GUI 生态 |
| 4 | 梯形/S 曲线加减速 | 支持线性加减速和可配置 S 曲线加减速（jerk 限制） |
| 5 | 结点速度优化 | 基于半角恒等式计算 junction angle，实现平滑过渡 |
| 6 | DSS 动态步进缩放 | 通过过采样实现超高步进频率，突破硬件定时器限制 |
| 7 | 运动学 HAL | 运动学独立于核心代码，支持自定义机器类型 |
| 8 | 工具 HAL | 支持主轴 PWM/继电器/VFD Modbus、激光 PPI/PWM、等离子 THC、绣花步进等 |
| 9 | 模块事件系统 | 核心代码预留 Hook 点，可挂载多个监听器扩展功能 |
| 10 | 背隙补偿 | 运动方向反转时自动补偿机械背隙 |
| 11 | G39 H-Mapping | 非平面表面高度映射补偿 |
| 12 | Flash EEPROM 仿真 | STM32 等无内置 EEPROM 的 MCU 上用 Flash 页模拟 |
| 13 | Web 配置工具 | 在线配置生成器，支持引脚映射与参数配置 |
| 14 | 原子操作原语 | 类 C11 的原子 CAS 和信号量，保证 ISR 安全 |

---

## 硬件架构

### 系统组成图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         µCNC 固件架构                                │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    G-Code 协议层                              │   │
│  │  ┌──────────┐  ┌──────────┐  ┌───────────┐  ┌───────────┐  │   │
│  │  │ grbl_     │  │ parser   │  │ grbl_     │  │ grbl_     │  │   │
│  │  │ stream    │→ │ (G-Code  │→ │ settings  │  │ protocol  │  │   │
│  │  │ (UART/    │  │  解析器)  │  │ (参数存储) │  │ (状态报告) │  │   │
│  │  │  USB)     │  │          │  │           │  │           │  │   │
│  │  └──────────┘  └──────────┘  └───────────┘  └───────────┘  │   │
│  └───────────────────────────┬─────────────────────────────────┘   │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    运动控制层                                 │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │   │
│  │  │ motion_      │→ │ planner      │→ │ interpolator │       │   │
│  │  │ control      │  │ (轨迹规划器)  │  │ (插补器)      │       │   │
│  │  │ (运动控制)    │  │              │  │              │       │   │
│  │  │ - 直线/圆弧   │  │ - 结点速度    │  │ - 加减速曲线  │       │   │
│  │  │ - 回零       │  │ - 加减速规划  │  │ - DSS 过采样  │       │   │
│  │  │ - 背隙补偿    │  │ - 缓冲区管理  │  │ - 步脉冲生成  │       │   │
│  │  └──────────────┘  └──────────────┘  └──────┬───────┘       │   │
│  └─────────────────────────────────────────────┼───────────────┘   │
│                                                  ▼                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    HAL 硬件抽象层                             │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │   │
│  │  │ MCU HAL  │  │ 运动学    │  │ 工具 HAL │  │ IO HAL   │    │   │
│  │  │ (定时器/  │  │ HAL      │  │ (主轴/   │  │ (限位/   │    │   │
│  │  │  GPIO/   │  │ (坐标↔   │  │  激光/   │  │  探针/   │    │   │
│  │  │  UART/   │  │  步数    │  │  等离子)  │  │  急停)   │    │   │
│  │  │  Flash)  │  │  变换)   │  │          │  │          │    │   │
│  │  └────┬─────┘  └──────────┘  └──────────┘  └──────────┘    │   │
│  └───────┼─────────────────────────────────────────────────────┘   │
│          ▼                                                         │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │              物理硬件层 (MCU + 外设)                       │      │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐        │      │
│  │  │AVR  │ │ESP32│ │STM32│ │RP2040│ │SAMD │ │LPC  │        │      │
│  │  │Mega │ │S3/C3│ │F1/F4│ │/2350│ │21   │ │176x │        │      │
│  │  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘        │      │
│  └──────────────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
```

### 硬件清单（以 STM32F103 典型配置为例）

| 组件 | 型号/规格 | 接口 | 说明 |
|------|----------|------|------|
| MCU | STM32F103C8T6 | — | 72MHz Cortex-M3, 64KB Flash, 20KB SRAM |
| 步进驱动 | A4988 / TMC2209 | GPIO (STEP/DIR/EN) | 每轴 3 个引脚 |
| 限位开关 | 机械/光电 | GPIO (上拉输入) | 每轴 2 个（最小/最大） |
| 急停按钮 | 常闭型 | GPIO (上拉输入) | ACTIVE_LOW |
| 主轴 | PWM 调速 | TIM PWM | 频率 ~1kHz |
| 串口 | USB-TTL / USB CDC | UART / USB | G-Code 通信 |
| EEPROM | 内部 Flash 仿真 | — | 1-2KB 设置存储 |
| 冷却 | 继电器/MOSFET | GPIO | 冷却液/雾冷 |

### 引脚映射（STM32F103 Blue Pill 示例）

| 功能 | 引脚 | 方向 | 说明 |
|------|------|------|------|
| STEP0 (X) | PA0 | 输出 | X 轴步进脉冲 |
| DIR0 (X) | PA1 | 输出 | X 轴方向 |
| STEP1 (Y) | PA2 | 输出 | Y 轴步进脉冲 |
| DIR1 (Y) | PA3 | 输出 | Y 轴方向 |
| STEP2 (Z) | PA4 | 输出 | Z 轴步进脉冲 |
| DIR2 (Z) | PA5 | 输出 | Z 轴方向 |
| EN (全部) | PA6 | 输出 | 步进使能（低有效） |
| LIMIT_X | PB0 | 输入 | X 轴限位 |
| LIMIT_Y | PB1 | 输入 | Y 轴限位 |
| LIMIT_Z | PB2 | 输入 | Z 轴限位 |
| ESTOP | PB10 | 输入 | 急停 |
| SPINDLE_PWM | PA7 | 输出 (PWM) | 主轴调速 |
| COOLANT_FLOOD | PB11 | 输出 | 水冷继电器 |
| UART_TX | PA9 | 输出 | 串口发送 |
| UART_RX | PA10 | 输入 | 串口接收 |

---

## 固件架构

### 文件结构表

| 路径 | 大小 | 职责 |
|------|------|------|
| `uCNC/src/cnc.c` | 30KB | 主控循环：初始化、G-Code 解析调度、告警处理、实时命令 |
| `uCNC/src/core/planner.c` | 19KB | 轨迹规划器：结点速度计算、加减速曲线规划、环形缓冲区 |
| `uCNC/src/core/interpolator.c` | 36KB | 插补器：线性/S 曲线加减速、DSS 过采样、步脉冲生成 |
| `uCNC/src/core/motion_control.c` | 37KB | 运动控制：直线/圆弧分段、背隙补偿、回零、H-Mapping |
| `uCNC/src/core/parser.c` | 84KB | G-Code 解析器：词法分析、状态机、模态组管理 |
| `uCNC/src/core/parser_expr.c` | 33KB | G-Code 表达式计算：参数 #、算术运算 |
| `uCNC/src/core/io_control.c` | 46KB | IO 控制：限位/探针/急停轮询、步进使能管理 |
| `uCNC/src/hal/kinematics/kinematic_*.c` | 1-13KB | 运动学变换：Cartesian/CoreXY/Delta/SCARA 等 |
| `uCNC/src/hal/mcus/stm32f1x/mcu_stm32f1x.c` | 40KB | STM32F1 HAL：定时器/GPIO/UART/Flash EEPROM |
| `uCNC/src/hal/mcus/esp32/mcu_esp32.c` | 13KB | ESP32 HAL：FreeRTOS 任务、GPIO、定时器 |
| `uCNC/src/hal/mcus/rp2040/mcu_rp2040.c` | 23KB | RP2040 HAL：PIO 步进生成、GPIO、定时器 |
| `uCNC/src/hal/tools/tools/*.c` | 2-15KB | 工具驱动：主轴/激光/等离子/绣花 |
| `uCNC/src/interface/grbl_protocol.c` | 30KB | Grbl 协议：状态报告、实时命令处理 |
| `uCNC/src/interface/grbl_settings.c` | 19KB | 参数管理：EEPROM 读写、设置校验 |
| `uCNC/src/interface/grbl_stream.c` | 10KB | 串口流：UART/USB 数据收发 |
| `uCNC/src/module.c` | 2.5KB | 模块系统：事件钩子注册与调用 |

### 模块职责与数据流

```
G-Code 文本 → grbl_stream → parser → motion_data_t
                                         ↓
                                    motion_control (mc_line)
                                         ↓
                                    planner (planner_add_line)
                                    ┌─────┴─────┐
                                    │ 规划缓冲区  │ ← 结点速度重算
                                    └─────┬─────┘
                                         ↓
                                    interpolator (itp_run)
                                    ┌─────┴─────┐
                                    │ 插补段缓冲  │ ← 加减速曲线
                                    └─────┬─────┘
                                         ↓
                                    STEP ISR (Bresenham)
                                         ↓
                                    MCU HAL → GPIO STEP/DIR
                                         ↓
                                    步进电机驱动器
```

---

## 核心代码深度分析

### 1. 系统初始化 — `cnc_init()`

系统启动时，`cnc_init()` 按严格顺序初始化所有子系统。这段代码揭示了 µCNC 的分层架构设计。

```c
// 文件: uCNC/src/cnc.c

#define LOOP_STARTUP_RESET 0
#define LOOP_UNLOCK 1
#define LOOP_RUNNING 2
#define LOOP_FAULT 3
#define LOOP_REQUIRE_RESET 4
#define LOOP_DO_RESET 5

typedef struct
{
    volatile uint16_t exec_state;
    uint8_t loop_state;
    volatile uint8_t rt_cmd;
    volatile uint8_t feed_ovr_cmd;
    volatile uint8_t tool_ovr_cmd;
    volatile int8_t alarm;
} cnc_state_t;

static cnc_state_t cnc_state;
bool cnc_status_report_lock;

void cnc_init(void)
{
#ifdef FORCE_GLOBALS_TO_0
    memset(&cnc_state, 0, sizeof(cnc_state_t));
    cnc_status_report_lock = false;
#endif
    cnc_state.loop_state = LOOP_STARTUP_RESET;
    // initializes all systems
    mcu_init();                                                                    // 1. MCU 硬件初始化
    mcu_io_reset();                                                                // 2. IO 引脚初始状态
    io_enable_steppers(~g_settings.step_enable_invert);                            // 3. 禁用所有步进电机
    io_disable_probe();                                                            // 4. 禁用探针中断
    grbl_stream_init();                                                            // 5. 串口流初始化
    mod_init();                                                                    // 6. 模块系统初始化
    settings_init();                                                               // 7. 设置加载（EEPROM/Flash）
    itp_init();                                                                    // 8. 插补器初始化
    planner_init();                                                                // 9. 规划器初始化
#if TOOL_COUNT > 0
    tool_init();                                                                   // 10. 工具初始化
#endif
    if (g_settings.homing_enabled)
    {
        cnc_set_exec_state(EXEC_POSITION_MAYBE_LOST);                              // 11. 标记位置可能丢失
    }
}
```

**逐行解析：**

- **`LOOP_STARTUP_RESET` 等宏**：定义了主循环的 6 个状态。`exec_state` 使用 `volatile` 修饰，因为它会被 ISR 修改；`loop_state` 不需要 volatile，因为它仅在主循环中读写。
- **`cnc_state_t` 结构体**：集中管理 CNC 运行状态。`exec_state` 是位域，每一位代表一种执行状态（如 `EXEC_RUN`、`EXEC_HOLD`、`EXEC_ALARM`）；`rt_cmd` 存储实时命令（进给倍率、主轴倍率等），需要在中断中快速响应。
- **初始化顺序**：严格遵循依赖关系——MCU 必须先初始化（提供定时器/GPIO/UART 基础），然后是 IO（依赖 GPIO），再是串口（依赖 UART），最后是 CNC 业务逻辑层（依赖前面的所有基础）。这种自底向上的初始化是嵌入式系统的经典模式。
- **`io_enable_steppers(~g_settings.step_enable_invert)`**：启动时禁用所有步进电机。`step_enable_invert` 是用户配置的使能电平反转标志，取反操作确保无论配置如何，上电时电机都处于禁用状态——这是一个安全设计。
- **`EXEC_POSITION_MAYBE_LOST`**：如果启用了回零功能，上电后位置是未知的，必须标记此状态。用户必须执行回零操作后才能进行运动——这是 CNC 安全的核心原则。

### 2. 主运行循环 — `cnc_run()` 与 `cnc_parse_cmd()`

```c
// 文件: uCNC/src/cnc.c

void cnc_run(void)
{
    cnc_reset();                                                                   // 执行完整复位
    if (cnc_unlock(false) != UNLOCK_ERROR)
    {
        cnc_state.alarm = EXEC_ALARM_NOALARM;
    }

    cnc_state.loop_state = LOOP_RUNNING;
    for (;;)
    {
        cnc_parse_cmd();                                                           // 解析 G-Code 命令
        cnc_dotasks();                                                             // 执行实时任务

        int8_t alarm = cnc_state.alarm;
        if (alarm > EXEC_ALARM_NOALARM)
        {
            cnc_alarm(alarm);
        }

        switch (cnc_state.alarm)
        {
        case EXEC_ALARM_NOALARM:
            break;
        case -EXEC_ALARM_HARD_LIMIT:
        case -EXEC_ALARM_SOFT_LIMIT:
            io_enable_steppers(~g_settings.step_enable_invert);
            proto_feedback(MSG_FEEDBACK_1);
            cnc_state.loop_state = LOOP_REQUIRE_RESET;
            __FALL_THROUGH__
        case EXEC_ALARM_EMERGENCY_STOP:
            cnc_wait_for_reset();
            __FALL_THROUGH__
        case EXEC_ALARM_SOFTRESET:
            cnc_state.alarm = EXEC_ALARM_NOALARM;
            return;
        default:
            if (cnc_get_exec_state(EXEC_POSITION_MAYBE_LOST))
            {
                return;
            }
            break;
        }
    }
}

uint8_t cnc_parse_cmd(void)
{
    uint8_t error = STATUS_OK;
    if (grbl_stream_available())
    {
        uint8_t c = grbl_stream_peek();
        switch (c)
        {
        case OVF:
            grbl_stream_overflow_flush();
            error = STATUS_OVERFLOW;
            break;
        case EOL:
            grbl_stream_getc();
            break;
        default:
            error = parser_read_command();                                         // 调用 G-Code 解析器
            break;
        }
        cnc_exec_rt_commands();                                                    // 执行实时命令
        proto_error(error);                                                        // 发送状态响应
        if (error)
        {
            itp_sync();
            mc_sync_position();
            parser_sync_position();
        }
    }
    return error;
}
```

**逐行解析：**

- **`cnc_run()` 的无限循环**：这是整个固件的心脏。每次迭代先解析一条 G-Code 命令，再执行实时任务。`cnc_dotasks()` 内部会调用 `itp_run()` 驱动步进电机——这种"解析一步、执行一步"的设计保证了 G-Code 解析不会阻塞运动控制。
- **`__FALL_THROUGH__`**：C 语言的 switch fall-through 显式标注。硬限位/软限位告警会依次执行：禁用步进→发送反馈→要求复位→等待复位。这种级联处理确保告警状态下系统不会继续运动。
- **`cnc_parse_cmd()` 中的 `grbl_stream_peek()`**：先窥视一个字符而不消费它。如果是溢出标记（OVF）则清空缓冲区，如果是行尾（EOL）则跳过空行，否则交给解析器处理。这种"先看后处理"的模式避免了空行导致的无效解析。
- **错误处理中的 `itp_sync()` + `mc_sync_position()`**：当 G-Code 解析出错时，需要同步插补器、运动控制器和解析器的位置——三者各自维护位置状态，错误后必须对齐，否则后续运动会基于错误位置执行。

### 3. 实时任务调度 — `cnc_dotasks()`

```c
// 文件: uCNC/src/cnc.c

bool cnc_dotasks(void)
{
    cnc_io_dotasks();                                                              // 1. IO 轮询（限位/探针/急停）
    cnc_exec_rt_commands();                                                        // 2. 执行实时命令

    if (cnc_state.loop_state == LOOP_STARTUP_RESET)
    {
        return false;
    }

    if (cnc_has_alarm() || (cnc_state.loop_state >= LOOP_FAULT))
    {
        return !cnc_get_exec_state(EXEC_KILL);
    }

    if (!cnc_check_interlocking())                                                 // 3. 安全联锁检查
    {
        return !cnc_get_exec_state(EXEC_INTERLOCKING_FAIL);
    }

#ifndef ENABLE_ITP_FEED_TASK
    if (!cnc_lock_itp)                                                             // 4. 插补器互斥锁
    {
        cnc_lock_itp = 1;
        itp_run();                                                                 // 5. 驱动步进插补
        cnc_lock_itp = 0;
    }
#endif

#ifdef ENABLE_TOOL_PID_CONTROLLER
    tool_pid_update();                                                             // 6. 工具 PID 更新
#endif

#ifdef ENABLE_MAIN_LOOP_MODULES
    cnc_modules_dotasks();                                                         // 7. 模块任务
#endif

    return !cnc_get_exec_state(EXEC_KILL);
}
```

**逐行解析：**

- **IO 轮询优先**：`cnc_io_dotasks()` 放在最前面，确保限位开关和急停按钮的响应延迟最小化。µCNC 采用轮询而非纯中断方式检测限位——这在多轴同时运动时更可靠。
- **`cnc_lock_itp` 互斥锁**：当 `ENABLE_ITP_FEED_TASK` 未定义时，插补器在主循环中运行。`cnc_lock_itp` 防止 `mcu_rtc`（毫秒定时器回调）中的代码与主循环中的 `itp_run()` 产生竞态条件。这是一个简单的自旋锁——在单线程嵌入式环境中足够安全。
- **`ENABLE_ITP_FEED_TASK`**：定义此宏后，插补器运行在独立的 FreeRTOS 任务（ESP32）或定时器中断中，不再需要互斥锁。这是 ESP32 等支持 RTOS 的平台的推荐配置。

### 4. 轨迹规划器 — 结点速度计算 `planner_add_line()`

这是 µCNC 运动规划的核心算法。当两条运动路径在结点处交汇时，规划器需要计算一个安全的过渡速度——既要保证平滑（不急停），又要保证安全（不超速）。

```c
// 文件: uCNC/src/core/planner.c

void planner_add_line(motion_data_t *block_data)
{
#ifdef ENABLE_LINACT_PLANNER
    static float last_dir_vect[STEPPER_COUNT];
#endif

    uint8_t index = planner_data_write;
    float cos_theta = block_data->cos_theta;
    memset(&planner_data[index], 0, sizeof(planner_block_t));
    planner_data[index].dirbits = block_data->dirbits;
    planner_data[index].feed_conversion = block_data->feed_conversion;
    planner_data[index].main_stepper = block_data->main_stepper;
    planner_data[index].planner_flags.reg = block_data->motion_flags.reg;

    memcpy(planner_data[index].steps, block_data->steps, sizeof(planner_data[index].steps));

    // 计算归一化方向向量
#ifdef ENABLE_LINACT_PLANNER
    float inv_total_steps = 1.0f / (float)(block_data->full_steps);
    float dir_vect[STEPPER_COUNT];
    memset(dir_vect, 0, sizeof(dir_vect));

    for (uint8_t i = STEPPER_COUNT; i != 0;)
    {
        i--;
        if (planner_data[index].steps[i] != 0)
        {
            dir_vect[i] = inv_total_steps * (float)planner_data[index].steps[i];

            if (!planner_buffer_is_empty())
            {
                cos_theta += last_dir_vect[i] * dir_vect[i];                       // 点积累加
            }
            last_dir_vect[i] = dir_vect[i];
        }
        else
        {
            last_dir_vect[i] = 0;
        }
    }
#endif

    planner_data[index].feed_sqr = fast_flt_pow2(block_data->feed);
    planner_data[index].rapid_feed_sqr = fast_flt_pow2(block_data->max_feed);
    planner_data[index].acceleration = block_data->max_accel;

    float angle_factor = 1.0f;
    uint8_t prev = 0;

    if (!planner_buffer_is_empty())
    {
        prev = planner_buffer_prev(index);
    }
    else
    {
        cos_theta = 0;
    }

    cos_theta = CLAMP(0, cos_theta, 1.0f);

    // 计算结点速度
    if (cos_theta != 0 && !CHECKFLAG(block_data->motion_mode,
        PLANNER_MOTION_EXACT_STOP | MOTIONCONTROL_MODE_BACKLASH_COMPENSATION))
    {
        if (cos_theta != 1.0f)
        {
            if (cos_theta > 0)
            {
                // 半角恒等式: tan(θ/2) = sqrt((1-cosθ)/(1+cosθ))
                // 简化为: sqrt((1-cos²θ)/(1+cosθ))
                angle_factor = fast_flt_inv(1.0f + cos_theta);
                cos_theta = (1.0f - fast_flt_pow2(cos_theta));
                angle_factor *= fast_flt_sqrt(cos_theta);
            }

            // G64 角度因子（用户可配置的拐角平滑）
            float factor = ((!CHECKFLAG(block_data->motion_mode,
                PLANNER_MOTION_CONTINUOUS)) ? 0 : g_settings.g64_angle_factor);
            angle_factor = CLAMP(0, angle_factor - factor, 1);

            if (angle_factor < 1.0f)
            {
                float junc_feed_sqr = (1 - angle_factor);
                junc_feed_sqr = fast_flt_pow2(junc_feed_sqr);
                junc_feed_sqr *= planner_data[prev].feed_sqr;
                // 最大结点速度 = min(当前速度, 角度限制速度)
                planner_data[index].entry_max_feed_sqr =
                    MIN(planner_data[index].feed_sqr, junc_feed_sqr);
            }
        }
        else
        {
            // cosθ=1 表示同方向运动，结点速度取两者较小值
            planner_data[index].entry_max_feed_sqr =
                MIN(planner_data[index].feed_sqr, planner_data[prev].feed_sqr);
        }

        planner_recalculate();                                                     // 触发全局速度重算
    }

    planner_add_block();                                                           // 写入环形缓冲区
}
```

**逐行解析：**

- **方向向量计算**：`dir_vect[i] = inv_total_steps * steps[i]` 将每轴的步数归一化为方向向量。`full_steps` 是所有轴步数的总和，用作归一化基数。两个方向向量的点积就是夹角的余弦值 `cos_theta`。
- **`last_dir_vect` 静态变量**：使用 `static` 保存上一段运动的方向向量。这意味着规划器在调用之间保持状态——它需要知道"之前在朝哪个方向走"才能计算结点角度。
- **半角恒等式变换**：核心数学。`tan(θ/2) = sqrt((1-cosθ)/(1+cosθ))`，代码中为了避免除法，将其改写为 `sqrt((1-cos²θ)) * (1/(1+cosθ))`。当 θ→0（直线运动）时 tan(θ/2)→0，结点速度不限制；当 θ→90°（直角转弯）时 tan(θ/2)→1，结点速度必须降为 0。
- **`g64_angle_factor`**：G64 是 G-Code 中的拐角平滑指令。用户可以设置一个角度因子，允许在拐角处保留一定的速度——因子越大，拐角处允许的速度越高，但路径精度越低。这是精度与速度的权衡。
- **`entry_max_feed_sqr` 使用平方速度**：整个规划器使用速度的平方（`feed_sqr`）而非速度本身。因为加减速公式 `v² = v₀² + 2as` 中只需要平方速度，避免了在每个插补周期中调用 `sqrt()`——这在没有 FPU 的 MCU（如 AVR）上可以节省大量计算时间。
- **`planner_recalculate()`**：每次添加新块后，需要从最新块向前回溯，重新计算所有缓冲块的最大入口速度。这是因为新块可能改变了后续块的约束——经典的 Grbl 反向递推算法。

### 5. 规划器缓冲区管理 — 原子操作

```c
// 文件: uCNC/src/core/planner.c

static void planner_add_block(void)
{
    uint8_t index = planner_data_write;
    uint8_t blocks = planner_data_blocks;
#if TOOL_COUNT > 0
    if (!blocks)
    {
        g_planner_state.spindle_speed = planner_data[index].spindle;
        g_planner_state.state_flags.reg = planner_data[index].planner_flags.reg;
    }
#endif

    if (++index == PLANNER_BUFFER_SIZE)
    {
        index = 0;
    }
    planner_data_write = index;

    // 原子递增——防止 step ISR 中的读取与主循环写入产生竞态
    ATOMIC_CODEBLOCK
    {
        planner_data_blocks++;
    }
}

void planner_discard_block(void)
{
    uint8_t blocks = planner_data_blocks;
    if (!blocks)
    {
        return;
    }

    uint8_t index = planner_data_read;
    uint8_t prev_index = index;

    if (++index == PLANNER_BUFFER_SIZE)
    {
        index = 0;
    }

    // 同步进给率：当前块的入口速度继承上一块的入口速度
    planner_data[index].entry_feed_sqr = planner_data[prev_index].entry_feed_sqr;

    blocks--;
    // ... 后续更新 planner_data_read 和 planner_data_blocks
}
```

**逐行解析：**

- **`ATOMIC_CODEBLOCK`**：这是 µCNC 的原子操作宏。在 AVR 上它关闭全局中断；在 ARM 上使用 `__disable_irq()`/`__enable_irq()`；在 ESP32 上使用 FreeRTOS 临界区。`planner_data_blocks` 同时被主循环（写入方）和 step ISR（读取方）访问，递增操作必须原子完成，否则 ISR 可能在递增的中间状态读取到错误值。
- **环形缓冲区**：`planner_data[PLANNER_BUFFER_SIZE]` 是固定大小的环形数组。`planner_data_write` 和 `planner_data_read` 分别是写指针和读指针。当 `write == read` 时缓冲区为空；当 `write+1 == read` 时缓冲区已满。这种设计避免了动态内存分配——嵌入式系统中的标准做法。
- **进给率继承**：`planner_discard_block()` 中 `planner_data[index].entry_feed_sqr = planner_data[prev_index].entry_feed_sqr` 确保当一块被消费后，下一块的入口速度从上一块的实际入口速度继承——这是速度连续性的保证。

### 6. 插补器核心 — `itp_run()` 加减速曲线计算

插补器负责将规划器生成的运动块转换为实际步进脉冲序列。它的核心任务是计算加速段、匀速段和减速段的步数分界点。

```c
// 文件: uCNC/src/core/interpolator.c

#define INTERPOLATOR_DELTA_T (1.0f / INTERPOLATOR_FREQ)
#define INTERPOLATOR_DELTA_CONST_T \
    (MIN((1.0f / INTERPOLATOR_BUFFER_SIZE), \
     ((float)(0xFFFF >> DSS_MAX_OVERSAMPLING) / (float)F_STEP_MAX)))

void itp_run(void)
{
    static uint32_t accel_until = 0;       // 加速段结束步数
    static uint32_t deaccel_from = 0;      // 减速段开始步数
    static float junction_speed = 0;       // 匀速段速度
    static float feed_convert = 0;
    static float t_acc_integrator = 0;     // 加速时间积分器
    static float t_deac_integrator = 0;    // 减速时间积分器

    itp_segment_t *sgm = NULL;

    while (!itp_sgm_is_full())
    {
        if (cnc_get_exec_state(EXEC_ALARM))
        {
            return;
        }

        // 获取新的规划块
        if (itp_cur_plan_block == NULL)
        {
            if (planner_buffer_is_empty())
            {
                break;
            }
            itp_cur_plan_block = planner_get_block();

            // 初始化 Bresenham 参数
            step_t total_steps = itp_cur_plan_block->steps[itp_cur_plan_block->main_stepper];
            itp_blk_data[itp_blk_data_write].total_steps = total_steps << 1;

            for (uint8_t i = 0; i < STEPPER_COUNT; i++)
            {
                // Bresenham 误差累加器初始化
                itp_blk_data[itp_blk_data_write].errors[i] = total_steps;
                itp_blk_data[itp_blk_data_write].steps[i] =
                    itp_cur_plan_block->steps[i] << 1;
            }

            itp_needs_update = true;
        }

        uint32_t remaining_steps = itp_cur_plan_block->steps[itp_cur_plan_block->main_stepper];
        sgm = &itp_sgm_data[itp_sgm_data_write];
        memset(sgm, 0, sizeof(itp_segment_t));
        sgm->block = &itp_blk_data[itp_blk_data_write];

        float current_speed = fast_flt_sqrt(itp_cur_plan_block->entry_feed_sqr);

        // 暂停时强制减速
        if (cnc_get_exec_state(EXEC_STOPPING))
        {
            accel_until = remaining_steps;
            deaccel_from = remaining_steps;
            t_deac_integrator = INTERPOLATOR_DELTA_T;
            itp_needs_update = true;
        }
        else if (itp_needs_update)
        {
            itp_needs_update = false;
            float exit_speed_sqr = planner_get_block_exit_speed_sqr();
            float junction_speed_sqr = planner_get_block_top_speed(exit_speed_sqr);

            junction_speed = fast_flt_sqrt(junction_speed_sqr);
            float accel_inv = fast_flt_inv(itp_cur_plan_block->acceleration);

            accel_until = remaining_steps;
            deaccel_from = 0;

            if (junction_speed_sqr != itp_cur_plan_block->entry_feed_sqr)
            {
                // 加速距离: d = (v² - v₀²) / (2a)
                float accel_dist = ABS(junction_speed_sqr -
                    itp_cur_plan_block->entry_feed_sqr) * accel_inv;
                accel_dist = fast_flt_div2(accel_dist);
                accel_until -= floorf(accel_dist);

                float t = ABS(junction_speed - current_speed);
                t *= accel_inv;  // t = (v - v₀) / a

                if (t > INTERPOLATOR_DELTA_T)
                {
                    // 将加速时间切片为整数个插补周期
                    float slices_inv = fast_flt_inv(floorf(INTERPOLATOR_FREQ * t));
                    t_acc_integrator = t * slices_inv;

                    if ((junction_speed_sqr < itp_cur_plan_block->entry_feed_sqr))
                    {
                        t_acc_integrator = -t_acc_integrator;  // 减速时取负
                    }
                }
                else
                {
                    accel_until = remaining_steps;  // 太短，不加速
                }
            }
        }
        // ... 后续生成步进段并计算减速点
    }
}
```

**逐行解析：**

- **静态变量**：`accel_until`、`deaccel_from` 等使用 `static` 修饰，在多次 `itp_run()` 调用之间保持状态。这是因为一个规划块可能需要多个插补周期才能完成，状态必须在调用之间持久化。
- **`total_steps << 1`**：步数左移 1 位（乘以 2）。这是 Bresenham 算法的标准技巧——将误差累加器初始值设为 `total_steps`（而非 `total_steps/2`），步数翻倍后用 `errors[i] -= steps[i]` 判断是否需要输出脉冲。左移比乘法更快。
- **加速距离公式**：`d = (v² - v₀²) / (2a)` 是运动学基本公式。代码中用 `accel_inv = 1/a` 预计算倒数，再用乘法代替除法——在没有硬件除法器的 MCU（如 AVR）上这是关键优化。`fast_flt_div2` 是除以 2 的快速实现（可能通过减指数实现）。
- **时间切片**：`slices_inv = 1 / floor(INTERPOLATOR_FREQ * t)` 将加速时间 `t` 切分为整数个插补周期。每个周期的速度增量 = 总增量 × `slices_inv`。这种"整数切片"确保加速过程的每个插补周期都有确定的速度变化量，避免了浮点累积误差。
- **`INTERPOLATOR_DELTA_T`**：插补周期 = `1 / INTERPOLATOR_FREQ`。这是插补器生成步进段的时间粒度。频率越高，步进越平滑，但 CPU 负载越大。典型值在 1-10kHz 范围。
- **`INTERPOLATOR_DELTA_CONST_T`**：匀速段的最小时间间隔，受两个约束：缓冲区大小（`1/INTERPOLATOR_BUFFER_SIZE`）和 DSS 过采样限制（`0xFFFF >> DSS_MAX_OVERSAMPLING / F_STEP_MAX`）。后者确保定时器计数值不超过 16 位范围。
- **暂停强制减速**：当 `EXEC_STOPPING` 状态激活时，`accel_until = remaining_steps` 和 `deaccel_from = remaining_steps` 将加速段和匀速段都置零，整个剩余距离全部用于减速。这是安全暂停的实现——电机必须平滑减速到停止，不能突然停止。

### 7. 运动控制 — 直线段与 Bresenham `mc_line_segment()`

```c
// 文件: uCNC/src/core/motion_control.c

static uint8_t mc_line_segment(int32_t *step_new_pos, motion_data_t *block_data)
{
#ifdef ENABLE_LINACT_PLANNER
    block_data->full_steps = 0;
#endif
    uint32_t max_steps = 0;

    for (uint8_t i = STEPPER_COUNT; i != 0;)
    {
        i--;
        int32_t s = step_new_pos[i] - mc_last_step_pos[i];                         // 计算步数差
        uint32_t steps = (uint32_t)ABS(s);
        block_data->steps[i] = (step_t)steps;

#ifdef ENABLE_LINACT_PLANNER
        block_data->full_steps += steps;                                           // 累加总步数
#endif

        if (max_steps < steps)
        {
            max_steps = steps;                                                     // 找出主导轴
#ifdef MOTION_NON_UNIFORM
            block_data->main_stepper = i;
#endif
        }
    }

    if (!max_steps)
    {
        return STATUS_OK;                                                          // 无运动
    }

    if (!mc_checkmode)
    {
#ifdef ENABLE_BACKLASH_COMPENSATION
        // 检测方向反转——背隙补偿
        uint8_t inverted_steps = mc_last_dirbits ^ block_data->dirbits;
        if (inverted_steps)
        {
            motion_data_t backlash_block_data = {0};
            memcpy(&backlash_block_data, block_data, sizeof(motion_data_t));
            memset(backlash_block_data.steps, 0, sizeof(backlash_block_data.steps));
            backlash_block_data.feed = backlash_block_data.max_feed;               // 背隙补偿用最大速度

            SETFLAG(backlash_block_data.motion_mode,
                MOTIONCONTROL_MODE_BACKLASH_COMPENSATION);
            max_steps = 0;
            for (uint8_t i = STEPPER_COUNT; i != 0;)
            {
                i--;
                if (inverted_steps & (1 << i))
                {
                    backlash_block_data.steps[i] = g_settings.backlash_steps[i];   // 补偿步数
                    if (max_steps < backlash_block_data.steps[i])
                    {
                        max_steps = backlash_block_data.steps[i];
                        backlash_block_data.main_stepper = i;
                    }
                }
            }

            while (planner_buffer_is_full())
            {
                if (!cnc_dotasks())
                {
                    return STATUS_CRITICAL_FAIL;
                }
            }
            planner_add_line(&backlash_block_data);                                // 先补偿背隙
        }
#endif
        // ... 然后添加实际运动
    }
    return STATUS_OK;
}
```

**逐行解析：**

- **主导轴（main_stepper）**：`max_steps` 是所有轴中步数最多的轴的步数。Bresenham 算法以主导轴为基准，其他轴通过误差累加器决定何时输出脉冲。例如 X 轴 1000 步、Y 轴 500 步，则 X 是主导轴，Y 每 2 步输出 1 个脉冲。
- **`full_steps` 累加**：所有轴步数的总和，用于规划器中方向向量的归一化计算。
- **背隙补偿**：`mc_last_dirbits ^ block_data->dirbits` 用异或检测哪些轴改变了方向。反转方向的轴需要先走 `backlash_steps[i]` 步来消除机械间隙（齿轮/丝杠的背隙）。补偿运动以最大速度执行，因为它不参与加工路径。
- **`while (planner_buffer_is_full())` 阻塞**：当规划缓冲区满时，运动控制必须等待插补器消费块。`cnc_dotasks()` 在等待期间持续执行——确保步进电机不会停顿，IO 不会被忽略。这是嵌入式系统中"阻塞等待 + 后台处理"的经典模式。

### 8. 笛卡尔运动学 — 坐标变换 `kinematic_cartesian.c`

```c
// 文件: uCNC/src/hal/kinematics/kinematic_cartesian.c

#if (KINEMATIC == KINEMATIC_CARTESIAN)

void kinematics_init(void)
{
    // 笛卡尔运动学无需初始化
}

// 逆运动学：机器坐标 → 步数
void kinematics_apply_inverse(float *axis, int32_t *steps)
{
    for (uint8_t i = 0; i < AXIS_COUNT; i++)
    {
        steps[i] = (int32_t)lroundf(g_settings.step_per_mm[i] * axis[i]);
    }
}

// 正运动学：步数 → 机器坐标
void kinematics_apply_forward(int32_t *steps, float *axis)
{
    for (uint8_t i = 0; i < AXIS_COUNT; i++)
    {
        axis[i] = (((float)steps[i]) / g_settings.step_per_mm[i]);
    }
}

uint8_t kinematics_home(void)
{
    float target[AXIS_COUNT];
    uint8_t error = STATUS_OK;

    // Z 轴先回零——避免撞坏工件
#if AXIS_Z_HOMING_MASK != 0
    error = mc_home_axis(AXIS_Z_HOMING_MASK, LINACT2_LIMIT_MASK);
    if (error != STATUS_OK)
        return error;
#endif

    // X/Y 轴回零
#ifndef ENABLE_XY_SIMULTANEOUS_HOMING
#if AXIS_X_HOMING_MASK != 0
    error = mc_home_axis(AXIS_X_HOMING_MASK, LINACT0_LIMIT_MASK);
    if (error != STATUS_OK)
        return error;
#endif
#if AXIS_Y_HOMING_MASK != 0
    error = mc_home_axis(AXIS_Y_HOMING_MASK, LINACT1_LIMIT_MASK);
    if (error != STATUS_OK)
        return error;
#endif
#else
    // X/Y 同时回零
    error = mc_home_axis(AXIS_X_HOMING_MASK | AXIS_Y_HOMING_MASK,
        LINACT0_LIMIT_MASK | LINACT1_LIMIT_MASK);
#endif

    // 设置原点位置
#ifdef SET_ORIGIN_AT_HOME_POS
    memset(target, 0, sizeof(target));
#else
    for (uint8_t i = AXIS_COUNT; i != 0;)
    {
        i--;
        target[i] = (!(g_settings.homing_dir_invert_mask & (1 << i)) ?
            0 : g_settings.max_distance[i]);
    }
#endif

    itp_reset_rt_position(target);
    return error;
}

bool kinematics_check_boundaries(float *axis)
{
    if (!g_settings.soft_limits_enabled || cnc_get_exec_state(EXEC_HOMING))
    {
        return true;
    }

    for (uint8_t i = AXIS_COUNT; i != 0;)
    {
        i--;
        if (g_settings.max_distance[i])
        {
            float value = axis[i];
            if (value > g_settings.max_distance[i] || value < 0)
            {
                return false;
            }
        }
    }
    return true;
}

#endif
```

**逐行解析：**

- **逆/正运动学**：笛卡尔运动学的变换极其简单——`steps = step_per_mm × mm`。但对于 Delta 或 SCARA 机器，这里会是非线性三角函数变换。µCNC 通过 `KINEMATIC` 宏在编译时选择运动学实现，核心代码完全不变。
- **`lroundf`**：使用 `lroundf`（四舍五入到 long）而非直接 `(int32_t)` 强制转换。强制转换是截断的，会导致系统性偏差——例如 1.9 步被截断为 1 步，长期累积导致位置漂移。
- **回零顺序**：Z 轴先回零是 CNC 的安全规范——Z 轴抬起后，X/Y 轴运动不会撞到工件。`AXIS_Z_HOMING_MASK` 等宏在配置文件中定义，如果某轴没有限位开关则跳过。
- **`SET_ORIGIN_AT_HOME_POS`**：如果定义此宏，回零位置就是原点 (0,0,0)；否则根据 `homing_dir_invert_mask` 判断回零方向，原点可能在 `max_distance` 处（如机床的原点在右后上角）。
- **软限位检查**：`kinematics_check_boundaries()` 在每条 G-Code 运动指令前被调用。回零过程中（`EXEC_HOMING` 状态）跳过检查——因为回零时机器需要触碰限位开关，这通常超出了 `max_distance` 的设定范围。

### 9. STM32F1 HAL — Flash EEPROM 仿真与 UART ISR

```c
// 文件: uCNC/src/hal/mcus/stm32f1x/mcu_stm32f1x.c

#if (MCU == MCU_STM32F1X)

// Flash EEPROM 仿真——STM32F1 没有内置 EEPROM
// 利用 Flash 最后一页存储设置，使用按位取反编码
#define READ_FLASH(ram_ptr, flash_ptr) (*ram_ptr = ~(*flash_ptr))
#define WRITE_FLASH(flash_ptr, ram_ptr) (*flash_ptr = ~(*ram_ptr))

static uint8_t stm32_flash_page[FLASH_PAGE_SIZE];
static uint16_t stm32_flash_current_page;
static bool stm32_flash_modified;

static volatile uint32_t mcu_runtime_ms;  // 毫秒计数器

// UART ISR——接收 G-Code，发送状态报告
void MCU_SERIAL_ISR(void)
{
    if (COM_UART->SR & USART_SR_RXNE)                                             // 接收寄存器非空
    {
        uint8_t c = COM_INREG;
        if (mcu_com_rx_cb(c))                                                      // 协议层回调
        {
            if (!BUFFER_TRY_ENQUEUE(uart_rx, &c))                                  // 写入环形缓冲区
            {
                STREAM_OVF(c);                                                     // 缓冲区溢出处理
            }
        }
    }

    if ((COM_UART->SR & USART_SR_TXE) && (COM_UART->CR1 & USART_CR1_TXEIE))       // 发送寄存器空且中断使能
    {
        uint8_t c = 0;
        if (!BUFFER_TRY_DEQUEUE(uart_tx, &c))                                      // 从发送缓冲区取数据
        {
            COM_UART->CR1 &= ~(USART_CR1_TXEIE);                                  // 缓冲区空，禁用发送中断
            return;
        }
        COM_OUTREG = c;                                                            // 写入发送寄存器
    }
}

// USB CDC 中断（STM32F103 内置 USB）
void USB_LP_CAN1_RX0_IRQHandler(void)
{
    tusb_cdc_isr_handler();
}

// 舵机定时器初始化
void servo_timer_init(void)
{
    RCC->SERVO_TIMER_ENREG |= SERVO_TIMER_APB;                                    // 使能定时器时钟
    SERVO_TIMER_REG->CR1 = 0;                                                      // 禁用定时器
    SERVO_TIMER_REG->DIER = 0;                                                     // 禁用中断
    SERVO_TIMER_REG->PSC = (SERVO_CLOCK / 255000) - 1;                            // 预分频：255kHz
    SERVO_TIMER_REG->ARR = 255;                                                    // 自动重装载：255 计数
    SERVO_TIMER_REG->EGR |= 0x01;                                                  // 生成更新事件
}

#endif
```

**逐行解析：**

- **Flash EEPROM 仿真**：STM32F1 没有真正的 EEPROM，µCNC 用 Flash 最后一页模拟。`READ_FLASH`/`WRITE_FLASH` 宏使用按位取反编码（`~`）——这是因为擦除后 Flash 全为 0xFF，取反后为 0x00，写入时只需将 0x00 改为所需值的取反，不需要擦除即可修改（Flash 只能从 1 写到 0）。这是一种巧妙的 wear-leveling 技巧。
- **`stm32_flash_page[FLASH_PAGE_SIZE]`**：RAM 中的 Flash 页缓存。写操作先修改 RAM 缓存，标记 `stm32_flash_modified = true`，在合适时机（如设置保存命令）才将整个页写入 Flash。这减少了 Flash 擦写次数。
- **UART ISR 的发送中断管理**：`COM_UART->CR1 &= ~(USART_CR1_TXEIE)` 在发送缓冲区空时禁用 TXE 中断——如果不禁用，TXE 中断会持续触发（因为发送寄存器一直为空），导致 CPU 被中断风暴淹没。当有新数据要发送时再使能中断。这是嵌入式 UART 驱动的标准做法。
- **`mcu_com_rx_cb(c)` 回调**：每收到一个字节都调用协议层回调。回调返回 `true` 表示该字节属于主协议（G-Code），需要入队；返回 `false` 表示被其他模块消费（如自定义命令）。这种设计允许在不修改核心代码的情况下扩展串口功能。
- **舵机定时器**：`PSC = (SERVO_CLOCK / 255000) - 1` 将定时器频率设为 255kHz，`ARR = 255` 使计数器在 0-255 之间循环。每个计数代表 1/255 的舵机周期——通过在特定计数值设置/清除引脚来生成 PWM 脉冲，精度约 4μs。

---

## API 使用指南

### 通过串口发送 G-Code

µCNC 兼容 Grbl 协议，可通过任意串口终端发送 G-Code 指令：

```bash
# 连接串口（以 /dev/ttyUSB0 为例，波特率 115200）
# 查看状态
echo "?" > /dev/ttyUSB0

# 查看设置
echo "$$" > /dev/ttyUSB0

# 查看参数信息
echo "$I" > /dev/ttyUSB0

# 解锁
echo "$X" > /dev/ttyUSB0

# 回零
echo "$H" > /dev/ttyUSB0

# G-Code 运动指令
echo "G0 X10 Y10 F1000" > /dev/ttyUSB0    # 快速移动到 (10,10)
echo "G1 X20 Y20 F500" > /dev/ttyUSB0     # 直线插补到 (20,20)，进给 500mm/min
echo "G2 X10 Y10 I-5 J0 F200" > /dev/ttyUSB0  # 顺时针圆弧插补

# 主轴控制
echo "M3 S1000" > /dev/ttyUSB0            # 主轴正转，1000 RPM
echo "M5" > /dev/ttyUSB0                  # 主轴停止

# 冷却控制
echo "M8" > /dev/ttyUSB0                  # 开启水冷
echo "M9" > /dev/ttyUSB0                  # 关闭冷却
```

### Python 控制（使用 pyserial）

```python
import serial
import time

# 打开串口
ser = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)
time.sleep(2)  # 等待初始化

def send_gcode(cmd):
    """发送 G-Code 并等待响应"""
    ser.write(f"{cmd}\n".encode())
    time.sleep(0.1)
    while ser.in_waiting:
        response = ser.readline().decode().strip()
        print(f"  ← {response}")
        if response == 'ok':
            break

# 解锁
send_gcode("$X")

# 查看状态
send_gcode("?")

# 回零
send_gcode("$H")

# 设置步进脉冲/mm
send_gcode("$0=80")    # X 轴 80 步/mm
send_gcode("$1=80")    # Y 轴 80 步/mm
send_gcode("$2=400")   # Z 轴 400 步/mm

# 运动测试
send_gcode("G21")       # 毫米单位
send_gcode("G90")       # 绝对定位
send_gcode("G0 X0 Y0 Z0 F1000")  # 移动到原点
send_gcode("G1 X50 Y50 F500")    # 直线插补

ser.close()
```

### cURL 流式 G-Code 发送（通过串口转 TCP 网关）

```bash
# 假设使用 ser2net 将串口映射到 TCP 端口
# 发送单条 G-Code
echo -e "G0 X10 Y10\n" | curl telnet://192.168.1.100:2323

# 批量发送 G-Code 文件
cat part.nc | curl telnet://192.168.1.100:2323
```

---

## 编译与部署

### 环境搭建

| 工具 | 版本 | 用途 |
|------|------|------|
| Arduino IDE | 2.0+ | ESP32/STM32/RP2040 编译烧录 |
| PlatformIO | Core 6+ | 命令行编译，支持所有平台 |
| GNU Arm Toolchain | 10-2021-q4 | STM32 裸机编译 |
| Make | 4.0+ | AVR/STM32 Makefile 构建 |
| Python | 3.8+ | 配置脚本与工具链 |

### 方式一：Arduino IDE（ESP32 示例）

1. 安装 Arduino IDE，添加 ESP32 板支持包
2. 克隆仓库并配置：
```bash
git clone https://github.com/Paciente8159/uCNC.git
cd uCNC
```
3. 使用 [Web 配置工具](https://paciente8159.github.io/uCNC-webconfig/) 生成 `cnc_config.h` 和 `cnc_hal_config.h`
4. 将配置文件放入 `uCNC/` 目录
5. 打开 `uCNC/uCNC.ino`，选择板卡型号，编译上传

### 方式二：PlatformIO

```ini
# platformio.ini
[env:esp32dev]
platform = espressif32
board = esp32dev
framework = arduino
build_flags = 
    -I uCNC
src_filter = +<*> +<../uCNC/src/>
```

```bash
pio run -e esp32dev        # 编译
pio run -t upload          # 烧录
pio device monitor -b 115200  # 串口监控
```

### 方式三：Makefile（STM32F1 裸机）

```bash
# 进入 STM32F1 makefile 目录
cd makefiles/stm32f1x

# 编辑 Makefile 设置工具链路径
export GCC_PATH=/path/to/arm-none-eabi-gcc/bin

# 编译
make

# 烧录（使用 ST-Link）
make flash
# 或使用 OpenOCD
openocd -f interface/stlink.cfg -f target/stm32f1x.cfg \
    -c "program uCNC.elf verify reset exit"
```

### 编译配置表

| 配置文件 | 作用 | 关键宏 |
|----------|------|--------|
| `cnc_config.h` | 机器配置 | `AXIS_COUNT`, `STEPPER_COUNT`, `KINEMATIC`, `PLANNER_BUFFER_SIZE` |
| `cnc_hal_config.h` | 硬件引脚映射 | `STEP0_BIT`, `DIR0_BIT`, `LIMIT_X_BIT`, `BAUD_RATE` |
| `cnc_machine_config.h` | 运动学参数 | `KINEMATIC_CARTESIAN`, `HOMING_CYCLE_0` |
| `sdkconfig.defaults` | ESP-IDF 配置 | `INTERPOLATOR_FREQ`, `F_STEP_MAX` |

### 烧录步骤（STM32F103 Blue Pill）

1. 连接 ST-Link V2 到 SWD 接口（SWCLK→DCLK, SWDIO→DIO, 3V3, GND）
2. 编译生成 `uCNC.bin`
3. 使用 STM32CubeProgrammer 烧录：
```bash
STM32_Programmer_CLI -c port=swd -w uCNC.bin 0x08000000 -v -rst
```
4. 连接 USB 串口（PA9/PA10），使用 UGS 或 CNCjs 连接
5. 发送 `$RST=*` 执行设置重置（首次必须）
6. 发送 `$H` 执行回零

---

## 项目亮点与适用场景

### 技术亮点

1. **极致的可移植性**：核心运动控制代码（planner、interpolator、motion_control）完全不依赖具体 MCU。移植到新平台只需实现 `mcu_*.c` 中的定时器、GPIO、UART 接口——约 15 个函数。目前已支持 6 大 MCU 家族。

2. **平方速度运算体系**：整个规划器和插补器使用 `v²` 而非 `v` 进行计算。加减速公式 `v² = v₀² + 2as` 不需要开方，仅在最终输出步进频率时调用一次 `sqrt()`。这在没有 FPU 的 AVR 上可节省 60% 以上的浮点运算时间。

3. **DSS 动态步进缩放**：当步进频率超过硬件定时器极限时，DSS 通过过采样技术（每 N 步输出 1 个脉冲的等效效果）实现超高频步进。`DSS_MAX_OVERSAMPLING` 定义过采样级别，每级翻倍——理论上可达 `F_STEP_MAX × 2^DSS_MAX_OVERSAMPLING`。

4. **模块事件系统**：核心代码中预埋了 `WEAK_EVENT_HANDLER` 弱符号钩子。用户可以在不修改核心代码的情况下，通过实现同名函数来拦截和扩展核心行为——如 `cnc_reset`、`cnc_dotasks`、`planner_pre_output` 等。这是 LinuxCNC 级别的可扩展性。

5. **编译时运动学选择**：通过 `#if (KINEMATIC == KINEMATIC_CARTESIAN)` 等条件编译，运动学代码在编译时就被选定。未使用的运动学实现完全不参与编译——零代码膨胀。这是嵌入式 C 语言中"零开销抽象"的经典实践。

### 适用场景

| 场景 | 推荐平台 | 说明 |
|------|----------|------|
| 桌面 CNC 雕刻机 | STM32F103/F407 | 3 轴 Cartesian，步进驱动 |
| 激光切割机 | ESP32 | 激光 PPI 模式，WiFi 远程控制 |
| 3D 打印机（修改） | RP2040 | CoreXY 运动学，PIO 步进生成 |
| Delta 机器人 | STM32F4 | Delta 运动学，6 轴支持 |
| 绣花机 | ESP32-S3 | 绣花工具模式，独立针头控制 |
| 等离子切割 | STM32F1 | THC 等离子高度控制 |
| 教学实验 | Arduino Mega | AVR 平台，丰富文档 |
| 工业自动化 | STM32H7 | 高性能，多轴联动 |

---

## 总结

µCNC 是一个在"紧凑高效"和"灵活模块化"之间取得了出色平衡的嵌入式 CNC 固件。它的 HAL 分层架构使得同一套运动控制核心可以运行在从 8 位 AVR 到 32 位 STM32H7 的广泛平台上——这在嵌入式 CNC 领域是极为罕见的。其轨迹规划器继承了 Grbl 的结点速度优化算法，并通过平方速度运算体系在无 FPU 的 MCU 上实现了高效计算；插补器的 DSS 过采样技术突破了硬件定时器的步进频率上限；模块事件系统则提供了接近 LinuxCNC 级别的可扩展性。

对于嵌入式开发者而言，µCNC 的代码是一座学习运动控制系统的宝库——从 G-Code 解析到步进脉冲生成，每一层都有值得深入研究的工程实践。对于 CNC 爱好者，它提供了一个可以直接使用、兼容 Grbl 生态、且支持几乎所有主流 MCU 的通用固件方案。

---

📝 作者：蔡浩宇（jun-chy）  
📅 日期：2026-07-14  
🔗 项目地址：[https://github.com/Paciente8159/uCNC](https://github.com/Paciente8159/uCNC)
