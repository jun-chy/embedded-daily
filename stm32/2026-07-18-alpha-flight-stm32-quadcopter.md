# Alpha-Flight 2.X：STM32F405 四旋翼飞控固件深度解析

> **日期**：2026-07-18 ｜ **分类**：STM32 / 飞行控制 / 无人机 ｜ **Stars**：1 ｜ **作者**：蔡浩宇（jun-chy）

## 项目链接

| 项目信息 | 内容 |
|---------|------|
| GitHub 地址 | [tephya/Alpha-Flight-2.X](https://github.com/tephya/Alpha-Flight-2.X) |
| 作者 | tephya |
| 许可证 | 未声明（参考仓库 LICENSE） |
| 最近更新 | 2026-07-18 |
| 主要语言 | C |
| 代码规模 | ~27.3 MB（含 PCB 文档、原理图、BOM、数据手册） |
| 默认分支 | main |

## 项目简介

Alpha-Flight 2.X 是一套完整的 10 英寸四旋翼无人机开发方案，包含底层飞控固件、硬件选型与配套 PCB 工程。其技术亮点在于：基于 STM32F405 的高频级联 PID 控制环（微分先行 + 动态抗饱和）、双 ICM-42688P 经独立 SPI 总线冗余、TPS54560 的 6S 兼容电源（模拟/数字电源隔离）、ELRS/CRSF 协议支持与 SPI 黑匣子日志。当前已完成台架空桨验证与初步外场试飞（Angle Mode），正进行闭环姿态与 PID 精调。

### 核心特性一览

| 特性 | 说明 |
|------|------|
| 主控 | STM32F405，168 MHz，Cortex-M4F |
| 控制环 | 高频级联 PID，微分先行（Derivative on Measurement）+ 动态反向计算抗饱和（Back-Calculation） |
| IMU 冗余 | 双 ICM-42688P，独立 SPI 总线（SPI1/SPI3），TIM7 硬件定时触发 DMA 连读 |
| 电调协议 | DSHOT600，4 通道经 TIM1 DMA Burst 并行输出 |
| 遥控 | ELRS/CRSF，USART2 + DMA 循环模式，420000 bps |
| 定位 | GPS（ATGM336H）+ QMC5883P 磁力计航向，返航逻辑 RTH（待调参） |
| 电源 | TPS54560 6S 兼容，3.3V_Clean / 3.3V_Dirty 隔离 |
| 黑匣子 | SPI TF 卡日志，DMA 写盘，临界区防数据撕裂 |
| 安全 | 独立看门狗 IWDG、倾角保护、低压强制停机、failsafe |

## 硬件架构

### 系统组成图

```
                         ┌─────────────────────────────────────┐
                         │          STM32F405 (168MHz)          │
                         │  ┌────────────────────────────────┐  │
                         │  │   TIM7 ──► SPI1 DMA ──► IMU1    │  │
                         │  │   (2ms 触发)  (连读12B)  (ICM-42688P)│  │
                         │  │   TIM6 软中断 ──► PID 计算       │  │
                         │  │   TIM1 DMA Burst ──► DSHOT600×4  │  │
                         │  │   USART2 + DMA ──► ELRS/CRSF     │  │
                         │  │   SPI2 ──► TF卡(黑匣子)          │  │
                         │  │   I2C(SW) ──► QMC5883P 磁力计    │  │
                         │  │   UART4 ──► GPS(ATGM336H)        │  │
                         │  │   ADC1 ──► 电池电压               │  │
                         │  │   IWDG 看门狗 / Buzzer / LED     │  │
                         │  └────────────────────────────────┘  │
                         └──────┬──────────┬──────────┬─────────┘
                                │          │          │
                  ┌─────────────┘    ┌─────┘          └────────────┐
                  ▼                  ▼                             ▼
          ┌──────────────┐   ┌──────────────┐            ┌────────────────┐
          │ 4× DSHOT600   │   │ ELRS 接收机   │            │ 6S LiPo + TPS54560│
          │ PA8~PA11 TIM1 │   │ PA2/PA3 USART2│            │ 3.3V_Clean/Dirty │
          │ → 4× ESC/电机 │   │ CRSF 420kbps  │            │ 电源隔离         │
          └──────────────┘   └──────────────┘            └────────────────┘
                  │
                  ▼
          ┌──────────────────────┐
          │  双 ICM-42688P (冗余) │  SPI1(PC4 CS) / SPI3(PB3 CS)
          │  ±16g ACC / ±2000dps │  ACC 1600Hz / GYRO 3200Hz
          └──────────────────────┘
```

### 引脚配置（核心外设映射）

| 外设 | 引脚 | 总线/定时器 | 说明 |
|------|------|------------|------|
| DSHOT CH1~CH4 | PA8/PA9/PA10/PA11 | TIM1 CH1~4 | 4 路电调 PWM |
| IMU1 SPI | PA5/PA6/PA7 + PC4(CS) | SPI1 | 主 IMU，DMA 连读 |
| IMU1 INT | PA4 | GPIO 输入 | ICM 数据就绪中断 |
| IMU2 SPI | (SPI3) + PB3(CS) | SPI3 | 冗余 IMU |
| ELRS RX/TX | PA3/PA2 | USART2 | CRSF 遥控，DMA 循环 |
| GPS | — | UART4 | ATGM336H NMEA |
| 磁力计 | — | 软件 I2C | QMC5883P |
| TF 卡 | — | SPI2 + DMA | 黑匣子日志 |
| 电池电压 | — | ADC1_IN10 | VBAT 采样 |
| LED | PB9 | GPIO 推挽 | 状态指示 |
| Buzzer | — | GPIO/定时器 | 蜂鸣器告警 |

### 硬件清单表

| 模块 | 型号 | 关键参数 |
|------|------|---------|
| 主控 MCU | STM32F405RGT6 | 168 MHz Cortex-M4F，FPU |
| IMU ×2 | ICM-42688P | 6 轴，±16g / ±2000dps，SPI |
| 磁力计 | QMC5883P | 3 轴，I2C |
| GPS | ATGM336H | 北斗/GPS，UART NMEA |
| 电调协议 | DSHOT600 | 11-bit 油门 + CRC |
| 遥控 | ELRS 接收机 | CRSF 420000 bps |
| 电源 | TPS54560 | 6S 降压，3.3V 隔离 |
| 存储 | TF 卡 (SPI) | 黑匣子日志 |
| 电机 | 10 寸四旋翼 | — |

## 固件架构

### 文件结构表

| 路径 | 模块 | 职责 |
|------|------|------|
| `User/main.c` | 主程序 | 初始化序列、解锁流程、主循环时间片调度、安全收尾 |
| `Hardware/ICM_42688P.c` | IMU 驱动 | SPI1/SPI3 配置、TIM7 硬件触发、DMA 连读、数据解析 |
| `Hardware/DSHOT.c` | 电调输出 | DSHOT600 编码、TIM1 DMA Burst 4 通道并行发送 |
| `Hardware/elrs.c` | 遥控接收 | USART2 + DMA 循环、CRSF 状态机组帧、CRC8、11-bit 通道解码 |
| `Hardware/GPS.c` | GPS 解析 | NMEA 解析、返航方位/距离计算 |
| `Hardware/QMC5883P.c` | 磁力计 | 3 轴读取、高斯换算、航向计算 |
| `Hardware/Motor.c` | 电机映射 | 控制器输出 → 4 路油门混控 |
| `Hardware/Buzzer.c` | 蜂鸣器 | 告警音驱动 |
| `Hardware/TFCARD.c` | 黑匣子 | SPI + DMA 读写、FATFS 日志 |
| `Flight/` | 控制核心 | 姿态解算、PID 控制、保护逻辑、导航 |
| `Libraries/` | 固件库 | STM32F4 标准外设库 |
| `Passthrough/` | 电调调试 | 独立 ESC 调试通道 |

## 核心代码深度分析

以下代码均取自仓库 `main` 分支真实源码，逐段解析。

### 1. 主控制流程与初始化序列（`User/main.c`）

`main.c` 的注释清晰描述了 7 步控制流程：初始化 → ARM ESC → 等 GPS 搜星记录首航 → ARM 遥控 → 主循环 → 延时落盘 → 退出。

**初始化阶段**：

```c
/* Peripheral Init */
DSHOT4_Init();
USART3_Init();
Init_ADC1();
GPS_Init();
Protection_Init();

SD_SPI_Init();
SD_Init();
SD_SPI_DMA_Init();

BB_Init();
BB_BufferInit();

IIC_Init();
BUZZER_Init();
ELRS_Init();

/* IMU & New Architecture Init */
ICM_SPI_Init();
ICM_Init();
ICM_SPI1_DMA_Init();         /* 初始化 SPI DMA 通道 */

Control_SoftwareTask_Init(); /* 初始化控制层软件中断 */
ICM_TIM7_Trigger_Init();     /* 初始化 IMU 硬件触发定时器 */
QMC_Init();
```

**逐段解析**：初始化严格按依赖顺序——先底层外设（DSHOT/USART/ADC/GPS），再存储（SD + DMA），再黑匣子缓冲，最后是"新架构"：IMU SPI 与 DMA 通道就绪后，`Control_SoftwareTask_Init` 初始化控制层软件中断（TIM6），`ICM_TIM7_Trigger_Init` 配置 IMU 硬件触发定时器。这个顺序保证 DMA/中断向量在 TIM7 使能前已就位，避免首帧触发时 DMA 未就绪。

**ESC 安全握手**：

```c
/* ESC 安全握手：throttle=0 持续 3 秒 */
BUZZER_Start();
for (int i = 0; i < 3000; i++) {
    if(i == 999 || i == 2999) BUZZER_Stop();
    if(i == 1999) BUZZER_Start();
    DSHOT4_Send(0, 0, 0, 0);
    Delay_ms(1);
}
```

**逐段解析**：以 1ms 节拍持续 3 秒发送 4 路零油门，完成 ESC 启动握手（多数 DSHOT ESC 需要看到持续零油门才进入待机）。蜂鸣器在第 1s/2s/3s 节点切换，用声音节奏提示握手进度——这是无屏幕飞控常见的状态反馈手段。

**GPS 首航记录**：

```c
USART_ITConfig(UART4, USART_IT_RXNE, ENABLE);
while(!gps_home.valid){
    GPS_SetHome();
}
```

**逐段解析**：开启 UART4 接收中断，阻塞直到 GPS 搜到星并记录首航坐标 `gps_home`。`gps_home.valid` 是退出条件——返航（RTH）逻辑以这个原点为基准。注释说明此阶段必须等待，因为没有 home 点的 RTH 无意义。

**解锁锁定循环**（带倾角保护）：

```c
while(!arm_state){
    ELRS_Poll();
    if(CRSF_PARSE_FRAME() == 1){
        __disable_irq();
        memcpy(&crsf_data, &temp_data, sizeof(CRSF_Data_t));
        __enable_irq();
    }
    if(SysTick_ms - last_dshot >= 20){
        last_dshot = SysTick_ms;
        DSHOT4_Send(0, 0, 0, 0);
    }
    /* 此时 TIM7 尚未开启，采用原阻塞读取进行安全倾角检测 */
    ICM_ReadACCData(SPI1, &imu1_data);
    float r_acc = atan2f(imu1_data.ay, imu1_data.az) * 57.2958f;
    float p_acc = atan2f(-imu1_data.ax, sqrtf(imu1_data.ay*imu1_data.ay + imu1_data.az*imu1_data.az)) * 57.2958f;
    uint8_t tilt_ok = (fabsf(r_acc) < 30.0f && fabsf(p_acc) < 30.0f);

    if(IS_ARMED() && Thr_LDet() && tilt_ok){
        arm_state = Armed;
        BUZZER_Stop();
        TIM_SetCounter(TIM7, 0);
        TIM_Cmd(TIM7, ENABLE);
        USART_ITConfig(UART4, USART_IT_RXNE, ENABLE);
    }
    else if(IS_ARMED() && Thr_LDet() && !tilt_ok){
        if((SysTick_ms / 150) % 2){ BUZZER_Start(); GPIO_SetBits(GPIOB, GPIO_Pin_9); }
        else { BUZZER_Stop(); GPIO_ResetBits(GPIOB, GPIO_Pin_9); }
    }
    ...
}
```

**逐段解析（重点）**：解锁前持续 20ms 发零油门保持 ESC 待机。关键安全逻辑是**倾角检测**——用加速度计 atan2 算 roll/pitch 角，倾斜超 30° 则拒绝解锁（`tilt_ok`），防止在异常姿态下启动电机飞甩。三个条件全满足才解锁：遥控 ARM 开关、油门拉杆最低、倾角合格。解锁瞬间 `TIM_Cmd(TIM7, ENABLE)` 启动高频硬件触发链——这是从"低速轮询"切到"高频 DMA 中断驱动"的开关。注释点明此时 TIM7 未开，所以用阻塞读取 IMU（而非 DMA）做检测。蜂鸣器用不同周期（150/250/600ms）的闪烁节奏区分"倾角超限/油门未归零/待解锁"三种状态。

### 2. 主循环时间片轮询调度（`User/main.c`）

解锁后进入主循环，采用基于 `SysTick_ms` 的时间片轮询，把不同频率的任务分到不同周期。

```c
while(1){
    IWDG_Feed();        // 1s 要喂一次狗
    BB_Process();       // SD卡写入（消费）

    /* 接收遥控器数据 */
    ELRS_Poll();
    if(CRSF_PARSE_FRAME() == 1){
        __disable_irq();
        memcpy(&crsf_data, &temp_data, sizeof(CRSF_Data_t));
        __enable_irq();
    }

    /* 慢速 I2C 任务：每 20ms 读取磁力计并单向注入航向角更新 */
    if(SysTick_ms - last_qmc >= 20) {
        last_qmc = SysTick_ms;
        if(QMC_ReadRaw_3Axis(&qmc_data) == 0) {
            QMC_Raw2Gauss(&qmc_data);
            Attitude_CptYaw(&qmc_data, &att1);
            FlightController_UpdateYaw(att1.yaw * 57.2958f);
        }
    }
    ...
}
```

**逐段解析**：主循环最顶层是 `IWDG_Feed` 喂狗与 `BB_Process` 消费黑匣子写盘队列。CRSF 数据消费用 `__disable_irq/__enable_irq` 临界区保护 `memcpy`，防止被高频控制中断打断导致 `crsf_data` 数据撕裂。磁力计 20ms（50Hz）读一次，单向注入航向角——比姿态环慢，因为磁力计响应慢且易受电机干扰。

**安全防御任务（100ms）**：

```c
/* 安全防御任务：每 100ms 检视机身状态，电压检测与过流 */
if(SysTick_ms - last_failsafe >= 100){
    last_failsafe = SysTick_ms;

    /* 异常倾角紧急停机坠机保护 */
    if(Tilt_Check(fc.roll_meas, fc.pitch_meas)){
        FlightController_Reset();
        TIM_Cmd(TIM7, DISABLE);
        arm_state = Disarmed;
        break;
    }

    ReadVBAT();
    if(vbat <= 13.2f) /* 强制坠机以保护锂电池 */
    {
        FlightController_Reset();
        TIM_Cmd(TIM7, DISABLE);
        DSHOT4_Send(0, 0, 0, 0);
        arm_state = Disarmed;
        break;
    }

    Protection_Update();
    Protection_SetMode();
    Buzzer_Drive();
}
```

**逐段解析**：100ms 安全巡检有两道硬停机——倾角超限立即停 TIM7（停 IMU 触发即停 PID 输出）、 disarm 退出；电压低于 13.2V（约 3.3V/cell，2S 等效阈值视电池配置）强制零油门停机以保护锂电池不过放。这两条都不经过 PID，直接 `break` 跳出主循环进入落盘收尾，是"宁可摔机也不烧电池/伤人"的最后防线。

**黑匣子写盘（500ms，临界区防撕裂）**：

```c
if(SysTick_ms - last_slow >= 500){
    last_slow = SysTick_ms;
    uint32_t delta_time = SysTick_ms - last_time_32;
    last_time_32 = SysTick_ms;
    record.time_ms = (uint16_t)delta_time;

    /* 临界区保护关键计算：防止写盘拼装时被高频控制快照打断导致数据撕裂 */
    __disable_irq();
    record.angle_cdeg[0] = (int16_t)(att1.roll * 10000.0f);
    record.angle_cdeg[1] = (int16_t)(att1.pitch * 10000.0f);
    record.angle_cdeg[2] = (int16_t)(att1.yaw * 5000.0f);
    __enable_irq();
    record.target_cdeg[0] = (int8_t)fc.roll_target;
    record.target_cdeg[1] = (int8_t)fc.pitch_target;
    record.target_cdeg[2] = (int8_t)fc.yaw_target;
    record.motor[0] = motor_thr_data.m1;
    ...
    BB_Log(&record);
}
```

**逐段解析**：500ms 慢速任务把姿态（roll/pitch/yaw）以 cdeg（厘度×10000，即 1e-4 度）定点整数存入黑匣子——定点整数避免浮点日志的精度/兼容问题。临界区只包住 `att1` 的读取（会被高频控制环改写），目标值与电机值在临界区外拼装。注释解释：高频控制链已通过软件中断"越顶"（preempt），所以这里可以阻塞写盘而不影响实时控制。

### 3. DSHOT600 四通道 DMA Burst 输出（`Hardware/DSHOT.c`）

这是本项目最精彩的硬件驱动之一：用 TIM1 的 DMA Burst 模式，一次 update 事件并行写 CCR1~CCR4，实现 4 路 DSHOT600 硬件级同步输出，CPU 零介入。

```c
#define DSHOT_ARR			279		/* 168MHz / 280 = 600khz = DSHOT600*/
#define DSHOT_BIT_0			105		/* 37.5% duty = 625ns */
#define DSHOT_BIT_1 		210		/* 75% duty = 1250ns */
#define DSHOT_FRAME_LEN		18		/* 16 data bits + 2 zero padding */
#define DSHOT_CHANNELS		4

static uint16_t dshot_buf[DSHOT_FRAME_LEN * DSHOT_CHANNELS] = {0};

static uint16_t dshot_encode_frame(uint16_t throttle){
	uint16_t val = (throttle << 1) | 0;
	uint16_t crc = (val ^ (val >> 4) ^ (val >> 8)) & 0x0F;
	return (val << 4) | crc;
}
```

**逐段解析**：`DSHOT_ARR=279` 即周期 280 个时钟，168MHz/280=600kHz，正是 DSHOT600 的位速率。`DSHOT_BIT_0=105`（37.5% 占空）与 `DSHOT_BIT_1=210`（75% 占空）符合 DSHOT 规范。`dshot_encode_frame` 编码 16 位帧：11 位油门 + 1 位 telemetry + 4 位 CRC。CRC 用 DSHOT 标准的异或算法 `(val ^ (val>>4) ^ (val>>8)) & 0x0F`，`val = throttle << 1` 把 telemetry 位左移让出最低位。最终 `(val << 4) | crc` 拼成 16 位。

**DMA Buffer 填充**：

```c
static void DSHOT_FillBuffer(uint16_t t1, uint16_t t2, uint16_t t3, uint16_t t4){
	uint16_t frame[4];
	frame[0] = dshot_encode_frame(t1);
	frame[1] = dshot_encode_frame(t2);
	frame[2] = dshot_encode_frame(t3);
	frame[3] = dshot_encode_frame(t4);

	/* 布局： [CCR1_bit0, CCR2_bit0, CCR3_bit0, CCR4_bit0, CCR1_bit1...] */
	for(int bit = 0; bit < 16; bit++){
		int base = bit * DSHOT_CHANNELS;
		for(int ch = 0; ch < 4; ch++){
			dshot_buf[base + ch] = ((frame[ch] >> (15 - bit)) & 1 )
									? DSHOT_BIT_1 : DSHOT_BIT_0;
		}
	}
	/* 末尾 2 个 padding 周期，全通道输出 0 */
	for(int pad = 16; pad < 18; pad++){
		int base = pad * DSHOT_CHANNELS;
		for(int ch = 0; ch < 4; ch++)
			dshot_buf[base + ch] = 0;
	}
}
```

**逐段解析（重点）**：Buffer 布局是 TIM DMA Burst 的精髓——TIM1_UP 事件每次从内存连续读 4 个 halfword 写入 DMAR（映射到 CCR1~CCR4）。所以内存必须按"每个 bit 周期 4 个通道值"交错排列：`[CCR1_bit0, CCR2_bit0, CCR3_bit0, CCR4_bit0, CCR1_bit1, ...]`。外层循环遍历 16 个数据位，内层循环 4 通道，从 `frame[ch]` 取第 `(15-bit)` 位（MSB 先发）填 BIT_1/BIT_0。末尾 2 个全 0 padding 周期生成帧间间隔（DSHOT 帧间需要复位时间）。

**发送函数**：

```c
void DSHOT4_Send(uint16_t t1, uint16_t t2, uint16_t t3, uint16_t t4)
{
	t1 = thr_to_dshot(t1, arm_state);
	t2 = thr_to_dshot(t2, arm_state);
	t3 = thr_to_dshot(t3, arm_state);
	t4 = thr_to_dshot(t4, arm_state);

	DSHOT_FillBuffer(t1, t2, t3, t4);

	DMA_Cmd(DMA2_Stream5, DISABLE);
	while(DMA_GetCmdStatus(DMA2_Stream5) != DISABLE);

	DMA_ClearFlag(DMA2_Stream5,
			DMA_FLAG_TCIF5 | DMA_FLAG_HTIF5 | DMA_FLAG_TEIF5 |
			DMA_FLAG_DMEIF5 | DMA_FLAG_FEIF5);
	DMA_SetCurrDataCounter(DMA2_Stream5, DSHOT_FRAME_LEN * DSHOT_CHANNELS);
	DMA_Cmd(DMA2_Stream5, ENABLE);
}
```

**逐段解析**：`thr_to_dshot` 把控制器油门映射到 DSHOT 有效范围（0 停转，1~47 特殊命令，48~2047 实际转速），未解锁直接返回 0。发送前先 `DISABLE` DMA 并 `while` 等其真正停下，防止上一帧未发完时篡改 buffer。清 5 类 DMA 标志（传输完成/半传输/传输错误/直接模式错误/FIFO 错误）后重设计数器并启动。整个 4 通道 18 周期的输出由 DMA 硬件完成，CPU 仅负责填充与启停。

### 4. SPI IMU DMA 连读与软件中断触发（`Hardware/ICM_42688P.c`）

IMU 采样用"TIM7 硬件触发 → SPI1 DMA 连读 → DMA 完成中断 → 软件中断触发 PID"的纯硬件链路，2ms 一次，CPU 几乎零介入。

```c
void ICM_TIM7_Trigger_Init(void){
    TIM_TimeBaseInitTypeDef tim;
    /* 定时器配置: 84MHz / 84 = 1 MHz（1us），2000下即2ms */
    tim.TIM_Period = 2000 - 1;
    tim.TIM_Prescaler = 84 - 1;
    ...
    TIM_ITConfig(TIM7, TIM_IT_Update, ENABLE);
    nvic.NVIC_IRQChannel = TIM7_IRQn;
    nvic.NVIC_IRQChannelPreemptionPriority = 1;     // 硬件触发保持最高优先级
    NVIC_Init(&nvic);
    TIM_Cmd(TIM7, DISABLE);
}

void TIM7_IRQHandler(void)
{
    if(TIM_GetITStatus(TIM7, TIM_IT_Update) != RESET) {
        TIM_ClearITPendingBit(TIM7, TIM_IT_Update);
        /* 纯粹触发 DMA 连读，不掺杂任何控制层业务 */
        ICM_Start_DMA_Transfer();
    }
}
```

**逐段解析**：TIM7 配在 APB1（84MHz），预分频 84 得 1MHz（1µs/tick），周期 2000 即 2ms 触发一次（500Hz 采样）。TIM7 中断服务函数**只做一件事**——触发 DMA 连读，注释强调"不掺杂任何控制层业务"，保持触发源的纯粹性与低延迟。抢占优先级 1 保证采样触发不被慢速任务抢占。

**DMA 连读 12 字节**：

```c
void ICM_Start_DMA_Transfer(void)
{
    /* 0x0C 是 ACCEL_DATA_X1，Bit7=1 表示读 */
    spi1_dma_tx_buf[0] = 0x0C | 0x80;
    GPIO_ResetBits(GPIOC, GPIO_Pin_4);  // 拉低 CS 开启传输

    DMA_Cmd(DMA2_Stream3, DISABLE);
    DMA_Cmd(DMA2_Stream0, DISABLE);
    DMA_SetCurrDataCounter(DMA2_Stream3, 13);
    DMA_SetCurrDataCounter(DMA2_Stream0, 13);

    /* 先开 RX，再开 TX */
    DMA_Cmd(DMA2_Stream0, ENABLE);
    DMA_Cmd(DMA2_Stream3, ENABLE);
}
```

**逐段解析**：TX buffer 首字节 `0x0C | 0x80`——`0x0C` 是 ACCEL_DATA_X1 寄存器地址，`bit7=1` 表示读操作（ICM-42688P 协议）。13 字节 = 1 字节地址 + 12 字节数据（ACC×6 + GYRO×6）。拉低 CS 后**先开 RX 再开 TX**——这是 SPI 全双工 DMA 的关键顺序：RX 必须先就绪，否则 TX 发出后返回的数据会丢失。

**DMA 完成中断解析并触发 PID**：

```c
void DMA2_Stream0_IRQHandler(void)
{
    if (DMA_GetITStatus(DMA2_Stream0, DMA_IT_TCIF0) != RESET)
    {
        DMA_ClearITPendingBit(DMA2_Stream0, DMA_IT_TCIF0);
        GPIO_SetBits(GPIOC, GPIO_Pin_4); // 1. 传输结束，拉高 CS

        /* 2. 解析 DMA 搬运来的原始数据并转换单位 */
        int16_t raw_ax = (int16_t)(spi1_dma_rx_buf[1] << 8 | spi1_dma_rx_buf[2]);
        ...
        imu1_data.ax = raw_ay * LSB_ACC;
        imu1_data.ay = raw_ax * LSB_ACC;
        imu1_data.az = raw_az * LSB_ACC;
        imu1_data.gx = raw_gy * LSB_GYO;
        imu1_data.gy = raw_gx * LSB_GYO;
        imu1_data.gz = raw_gz * LSB_GYO;

        /* 数据准备好了，触发 TIM6 软件中断去算 PID */
        NVIC_SetPendingIRQ(TIM6_DAC_IRQn);
    }
}
```

**逐段解析（重点）**：DMA 传输完成中断里拉高 CS 结束 SPI 事务，然后把 12 字节原始数据拼成 16 位有符号整数，乘 LSB 系数转物理单位。注意 `ax/ay` 与 `gx/gy` 做了轴重映射（`imu1_data.ax = raw_ay`），适配 PCB 上 IMU 的安装朝向。最关键的是最后一行 `NVIC_SetPendingIRQ(TIM6_DAC_IRQn)`——通过置位 TIM6 中断挂起位，**用软件方式触发一个硬件中断**去跑 PID。这样 PID 计算在 TIM6 中断上下文里执行，优先级可控，与采样触发解耦，是嵌入式"硬件触发采样 + 软件中断计算"的经典设计模式。

### 5. CRSF 协议状态机与 DMA 环形缓冲（`Hardware/elrs.c`）

ELRS/CRSF 接收用 USART2 + DMA 循环模式，CPU 零介入搬运，主循环用追赶指针消费。

**DMA 循环接收**：

```c
usart.USART_BaudRate = 420000;
...
dma.DMA_Mode = DMA_Mode_Circular;
...
DMA_Init(DMA1_Stream5, &dma);
USART_DMACmd(USART2, USART_DMAReq_Rx, ENABLE);
USART_Cmd(USART2, ENABLE);
DMA_Cmd(DMA1_Stream5, ENABLE);
```

**逐段解析**：CRSF 标准波特率 420000 bps。DMA 设为 `DMA_Mode_Circular` 循环模式——DMA 到达缓冲区末尾自动回卷到开头，硬件持续把串口字节搬进 `dma_rx_buf`，全程无 CPU 介入，杜绝高频控制下串口数据阻塞丢失。

**追赶指针消费**：

```c
void ELRS_Poll(void)
{
    uint16_t counter = DMA_GetCurrDataCounter(DMA1_Stream5);
    uint16_t write_ptr = (DMA_BUF_SIZE - counter) % DMA_BUF_SIZE;

    while(read_ptr != write_ptr){
        uint8_t byte = dma_rx_buf[read_ptr];
        ELRS_StateMachine(byte);
        read_ptr = (read_ptr + 1) % DMA_BUF_SIZE;
    }
}
```

**逐段解析**：通过 `DMA_GetCurrDataCounter` 读 DMA 剩余传输量，反推硬件写到了哪里（`write_ptr`）。本地 `read_ptr` 追赶 `write_ptr`，把新字节逐个喂进状态机。这是 DMA 循环缓冲的经典"生产者（硬件 DMA）-消费者（软件轮询）"模型，无锁且高效。

**CRSF 帧状态机**：

```c
void ELRS_StateMachine(uint8_t byte)
{
    switch(state)
    {
        case 0:						/* 等Sync */
            if(byte == 0xC8) {
                if(frame_ready) break;	/* 上一帧还没被解析，丢弃新帧保平安 */
                state = 1;
                frame_buf[idx++] = 0xC8;
            }
            break;
        case 1: 					/* 读Len */
            if(byte > 62){			/* Len最大62，防止噪声导致数组越界 */
                state = 0; idx = 0; break;
            }
            state = 2;
            frame_buf[idx++] = byte;
            frame_len = byte + 2;   /* 总帧长 = Sync(1) + Len(1) + Len里的数值 */
            break;
        case 2:						/* 收数据 */
            frame_buf[idx++] = byte;
            if(idx == frame_len) {
                state = 0;
                frame_ready = 1;
                idx = 0;
            }
            break;
    }
}
```

**逐段解析**：三状态机——等 Sync(0xC8) → 读 Len → 收数据。两处防御：① 上一帧未消费（`frame_ready` 仍为 1）时新帧直接丢弃，防止半帧覆盖引发系统崩溃；② Len > 62 立即复位，防止噪声字节导致 `frame_buf` 数组越界。`frame_len = byte + 2` 因为总帧长 = Sync(1) + Len(1) + Len 字段值（含 Type+Payload+CRC）。

**11-bit 通道解码**：

```c
case 0x16:  /* 摇杆通道数据 */
{
    for(int i = 0; i < 16; i++){
        start_bit = i * 11;
        byte_index = start_bit / 8;
        offset = start_bit % 8;
        raw = (uint32_t)frame_buf[byte_index + 3]     /* 跳过 Sync + Len + Type (3Bytes) */
            | (uint32_t)frame_buf[byte_index + 4] << 8
            | (uint32_t)frame_buf[byte_index + 5] << 16;
        temp_data.channels[i] = (raw >> offset) & 0x7FF;
    }
    break;
}
```

**逐段解析（重点）**：CRSF 把 16 个通道每个 11 bit 紧凑打包成 22 字节位流。解码每个通道时算出其起始 bit（`i*11`），转成字节索引与位偏移。因为 11 bit 可能跨 2~3 字节边界，所以一次读 3 个字节拼成 32 位 `raw`，再右移 `offset` 位并用 `0x7FF`（11 个 1）掩码取出通道值。`+3` 是跳过帧头 Sync+Len+Type 三字节。这种位操作是协议解析的教科书实现。

**CRC8 查表校验**：

```c
static const uint8_t crc8_table[256] = { 0x00, 0xD5, 0x7F, ... };

uint8_t CRSF_CRC8(void){
    uint8_t crc = 0;
    for(int i = 2; i < frame_len - 1; i++)
        crc = crc8_table[crc ^ frame_buf[i]];
    return crc;
}
```

**逐段解析**：CRC 校验范围是 `frame_buf[2]` 到 `frame_buf[frame_len-2]`（Type + Payload，不含 Sync 与 Len，也不含末尾 CRC 字节）。用 256 字节查表代替逐位多项式计算，速度更快，适合嵌入式。只有 CRC 通过的帧才会更新 `temp_data`，脏数据被拒绝。

## API 使用指南

Alpha-Flight 是裸机固件，不提供 REST/RPC API，交互通过遥控器（CRSF）、串口（USART3 调试）、黑匣子日志（TF 卡）进行。

### 串口调试 / 日志读取

```bash
# 1. 连接飞控 USART3 调试串口（printf 重定向到 USART3）
stty -F /dev/ttyUSB0 115200 cs8 -cstopb -parenb
cat /dev/ttyUSB0   # 实时查看 printf 调试输出

# 2. 读取黑匣子日志（TF 卡 SPI 记录的飞行数据）
#    日志为二进制 BB_Frame_t 结构数组，需按结构解析
python3 - <<'PY'
import struct
# BB_Frame_t 示例布局（以 main.c 写入字段为准，按实际固件结构核对）
rec = struct.Struct('<H 3h 3b 4B')   # time_ms, angle_cdeg[3], target_cdeg[3], motor[4]
with open('blackbox.bin','rb') as f:
    data = f.read()
for i in range(0, len(data), rec.size):
    t, r, p, y, tr, tp, ty, m1, m2, m3, m4 = rec.unpack_from(data, i)
    print(f"t={t}ms roll={r/10000:.2f} pitch={p/10000:.2f} motor=[{m1},{m2},{m3},{m4}]")
PY
```

### CRSF 通道测试（Python，经 USB-CRSF 适配器）

```python
# 模拟 CRSF 帧发送，验证飞控通道解析
def crsf_crc8(data):
    crc = 0
    for b in data:
        crc = crc8_table[crc ^ b]
    return crc

# 构造 0x16 通道帧：16 通道 × 11 bit = 22 字节
channels = [992]*16  # CRSF 中点值约 992（11-bit 0..2047）
payload = bytearray(22)
for i, ch in enumerate(channels):
    bit = i * 11
    byte_idx = bit // 8
    off = bit % 8
    val = (ch & 0x7FF) << off
    for k in range(3):
        payload[byte_idx + k] |= (val >> (8*k)) & 0xFF
frame = bytes([0xC8, 24, 0x16]) + payload
frame += bytes([crsf_crc8(frame[2:])])
serial.write(frame)
```

## 编译与部署

### 环境搭建

| 依赖 | 版本 | 说明 |
|------|------|------|
| Keil MDK-ARM | 5.x | 主 IDE，工程文件 `Alpha.uvprojx` |
| ARM Compiler | V5/V6 | Cortex-M4F 支持 |
| STM32F4 标准外设库 | 1.8.x | `Libraries/` 目录 |
| ST-Link V2/V3 | — | 烧录调试 |
| 串口工具 | 任意 | USART3 调试输出 |

### 编译配置表

| 配置项 | 值 | 说明 |
|--------|-----|------|
| 目标芯片 | STM32F405RG | 168MHz Cortex-M4F |
| 晶振 | 8MHz HSE | PLL 倍频至 168MHz |
| 优化等级 | -O2 | 平衡速度与体积 |
| 浮点 | FPU Hard | 硬件单精度 FPU |
| C 标准库 | microlib | 减小体积 |
| 调试 | ST-Link | SWD |

### 烧录步骤

```bash
# 1. 用 Keil 打开 Alpha.uvprojx
# 2. 选择目标 STM32F405RG，配置 ST-Link
# 3. Build（F7）生成 .hex/.axf
# 4. Flash（F8）经 ST-Link 烧录

# 命令行烧录（ST-Link Utility）
ST-LINK_CLI -c SWD UR -P Alpha.hex -V -RST
```

烧录后流程：上电 → 蜂鸣器启动音 → ESC 握手（3s 零油门）→ 等 GPS 搜星 → 解锁（ARM 开关 + 油门最低 + 倾角 <30°）→ 起飞。

## 项目亮点与适用场景

**亮点**
- **纯硬件采样链路**：TIM7 → SPI DMA → DMA 中断 → 软件中断触发 PID，CPU 零介入采集，2ms 稳定。
- **TIM1 DMA Burst 四路并行 DSHOT600**：4 路电调硬件级同步输出，无逐通道软件开销。
- **双 IMU SPI 冗余架构**：独立 SPI 总线，单点失效不致坠机。
- **分级安全保护**：倾角保护、低压停机、IWDG 看门狗、failsafe，层层兜底。
- **完整硬件工程**：含原理图、PCB、BOM、数据手册，可复现。

**适用场景**
- 四旋翼飞控开发学习（PID、SPI DMA、DSHOT、CRSF 的工程参考）。
- STM32 标准外设库 + DMA 多通道并发实践。
- 嵌入式实时调度设计（时间片轮询 + 软件中断 + 临界区）。
- 电调协议与遥控协议解析实现参考。

## 总结

Alpha-Flight 2.X 是一份"硬件 + 固件 + 文档"三位一体的四旋翼方案。其工程价值在于把 STM32F405 的定时器/DMA/SPI 用到了极致：TIM7 硬件触发 SPI DMA 连读 IMU、TIM1 DMA Burst 并行输出 DSHOT600、USART2 DMA 循环接收 CRSF、TIM6 软件中断触发 PID——整条控制链路 CPU 介入极少，实时性靠硬件链路保证。对于想深入理解"裸机飞控如何用 DMA + 定时器 + 软件中断搭建高频控制环"的嵌入式开发者，`ICM_42688P.c`、`DSHOT.c`、`elrs.c`、`main.c` 是非常值得逐行研读的实战范本。项目当前仍在 PID 精调阶段，是观察一个真实飞控迭代过程的好样本。

---

📝 作者：蔡浩宇（jun-chy） ｜ 📅 日期：2026-07-18 ｜ 🔗 项目地址：https://github.com/tephya/Alpha-Flight-2.X
