# STM32 FOC 电机控制固件深度解析 — MESC_Firmware

> 📅 日期：2026-07-04 | 🏷️ 分类：STM32 | ⭐ Stars：257 | 👤 作者：蔡浩宇（jun-chy）

---

## 项目信息

| 属性 | 详情 |
|------|------|
| **GitHub 地址** | [davidmolony/MESC_Firmware](https://github.com/davidmolony/MESC_Firmware) |
| **作者** | David Molony (davidmolony) / Jens Kerrinnes |
| **许可证** | BSD-3-Clause |
| **最近更新** | 2026-07-03 |
| **主要语言** | C |
| **Forks** | 68 |
| **默认分支** | master |
| **Topics** | field-oriented-control, field-weakening, foc, mtpa, stm32 |

---

## 一、项目简介

MESC_Firmware（Molony Electronic Speed Controller）是一个开源的 STM32 无刷电机 FOC（磁场定向控制）固件库。它兼容所有带 FPU 的 STM32 目标，支持 BLDC（无刷直流电机）和 PMSM（永磁同步电机），实现了完整的电机控制链路：从 ADC 采样、Clarke/Park 变换、PI 闭环控制到 SVPWM 输出。

该项目的核心价值在于其自动测量与校准系统——无需手动查阅电机数据手册，固件可以自动测量电机的相电阻（R）、相电感（Ld/Lq）、磁链（flux linkage）、HFI 阈值和死区时间补偿，真正实现"接上电机就能跑"。

### 核心特性一览

| 特性 | 说明 |
|------|------|
| FOC 控制 | 完整 Clarke/Park 变换 + 双 PI 闭环（Id/Iq） |
| 多观测器 | 支持滑模观测器(SMO)、磁链观测器、HFI 高频注入 |
| 自动测量 | R/L/磁链/HFI 阈值/死区时间全自动测量 |
| 弱磁控制 | Field Weakening 扩展高速范围 |
| MTPA | 最大转矩每安培控制策略 |
| FreeRTOS | 基于 RTOS 的多任务架构 |
| CAN 通信 | 支持 CAN 总线遥测与控制 |
| USB CDC | 虚拟串口终端调试 |
| 多平台 | STM32F4 (F411CE) / STM32L4 (L431RC) |
| 开源终端 | 内置 TTerm 终端库，支持变量实时调节 |

---

## 二、硬件架构

### 系统组成图

```
              ┌─────────────────────────────────────────┐
              │              STM32 MCU                   │
              │  ┌───────────────────────────────────┐   │
              │  │  FreeRTOS  │  TTerm Terminal       │   │
              │  │  Tasks     │  (USB CDC / UART)    │   │
              │  ├───────────┼───────────────────────┤   │
              │  │  MESC FOC Core                    │   │
              │  │  ┌─────────┐  ┌──────────────┐    │   │
              │  │  │Clarke/   │  │  PI Control  │    │   │
              │  │  │Park Trans│  │  (Id/Iq loop)│    │   │
              │  │  └────┬────┘  └──────┬───────┘    │   │
              │  │       │              │             │   │
              │  │  ┌────▼──────────────▼───────┐    │   │
              │  │  │  Observer (SMO/Flux/HFI)  │    │   │
              │  │  └────────────┬──────────────┘    │   │
              │  └───────────────┼───────────────────┘   │
              │                  │                       │
              │  ┌───────────────▼───────────────────┐   │
              │  │  ADC (Injected) │ TIM1 (PWM/ADC)  │   │
              │  │  Iu/Iv/Iw/Vbus  │  SVPWM 6-ch     │   │
              │  └───────┬─────────┴────────┬────────┘   │
              └──────────┼──────────────────┼────────────┘
                         │                  │
          ┌──────────────┼──────────┐      │
          │              │          │      │
    ┌─────▼─────┐ ┌──────▼──┐ ┌─────▼───┐  │
    │ 电流采样   │ │ 电压采样 │ │  编码器  │  │
    │ Shunt+OpAmp│ │ Vbus分压 │ │  /Hall  │  │
    │ 3相        │ │          │ │         │  │
    └───────────┘ └──────────┘ └─────────┘  │
                                            │
              ┌─────────────────────────────▼─────┐
              │     3-Phase Inverter (MOSFET)      │
              │    ┌─────┬─────┬─────┐             │
              │    │  U  │  V  │  W  │  Phase      │
              │    │Bridge│Bridge│Bridge│ Output    │
              │    └──┬──┴──┬──┴──┬──┘             │
              └───────┼─────┼─────┼────────────────┘
                      │     │     │
                   ┌──▼─┐ ┌──▼─┐ ┌──▼─┐
                   │ M  │ │ M  │ │ M  │  BLDC / PMSM
                   │    │ │    │ │    │  电机
                   └────┘ └────┘ └────┘
```

### 引脚与外设配置

以下基于 MESC_L431RC 目标板的硬件初始化代码：

| 外设 | 用途 | 说明 |
|------|------|------|
| ADC1 (Injected) | 电流采样 | JDR1=Iu, JDR2=Iv, JDR3=Iw, JDR4=Vbus |
| ADC1 (Regular/DMA) | 电压/温度 | ADC_buffer[0-2]=Vu/Vv/Vw, [4-5]=油门, [6]=电机温度 |
| TIM1 | PWM 输出 | 6 通道互补 PWM + 死区时间，触发 ADC 注入转换 |
| CAN | 通信 | CAN 总线遥测与控制 |
| USART | 调试 | TTerm 终端交互 |
| USB | CDC | 虚拟串口 |

### 硬件清单

| 组件 | 型号/规格 | 说明 |
|------|-----------|------|
| MCU | STM32L431RC / STM32F411CE | 带 FPU 的 Cortex-M4 |
| 电流采样 | 低边分流电阻 + 运放 | 3 相电流检测 |
| 电压采样 | 电阻分压网络 | 母线电压检测 |
| 功率级 | 6× MOSFET | 3 相逆变桥 |
| 编码器 | 磁编码器/Hall | 位置反馈（可选） |
| 电源 | DC-DC 降压 | 逻辑供电 |

---

## 三、固件架构

### 文件结构

```
MESC_Firmware/
├── MESC_L431RC/              # STM32L431RC 目标板工程
│   ├── Core/
│   │   ├── Inc/
│   │   │   ├── MESC_L431.h          # L431 硬件引脚定义
│   │   │   ├── MX_FOC_IMS.h         # FOC 中断管理
│   │   │   └── main.h               # CubeMX 主头文件
│   │   └── Src/
│   │       ├── main.c               # 主程序入口
│   │       └── MESChw_setup.c       # 硬件初始化 + ADC 读取
│   └── Drivers/                     # STM32 HAL/CMSIS 驱动
│
├── MESC_F411CE/              # STM32F411CE 目标板工程
│   └── ... (同上结构)
│
├── MESC_RTOS/                # 核心固件库（跨平台）
│   ├── MESC/
│   │   ├── MESCinterface.c          # 主接口：测量/校准/变量注册
│   │   ├── MESCinterface.h          # 接口声明
│   │   ├── MESCmotor_state.h        # 电机状态机定义
│   │   ├── calibrate.c              # ADC 校准终端应用
│   │   ├── hfi.c                    # 高频注入无感控制
│   │   ├── task_LED.c               # LED 状态指示任务
│   │   └── MESCfoc.c                # FOC 核心算法（Clarke/Park/PI/SVPWM）
│   ├── Tasks/
│   │   ├── init.c                    # 系统初始化任务
│   │   ├── task_can.c               # CAN 通信任务
│   │   ├── task_cli.c               # CLI 命令行任务
│   │   └── task_overlay.c           # 调试叠加层
│   ├── AXIS/                        # 多轴支持
│   ├── Dash/                        # 仪表盘接口
│   └── TTerm/                       # 内置终端库
│       └── Core/
│           ├── TTerm.c              # 终端核心
│           ├── TTerm_var.c          # 变量管理系统
│           └── TTerm_AC.c           # 自动补全
│
└── Axis-Throttle/            # 油门手柄应用工程
    └── Core/Src/main.c              # 基于 STM32L432 的油门控制器
```

### 模块职责

| 模块 | 文件 | 职责 |
|------|------|------|
| 硬件抽象 | `MESChw_setup.c` | ADC 校准、电流/电压读取、增益计算 |
| FOC 核心 | `MESCfoc.c` | Clarke/Park 变换、PI 闭环、SVPWM、观测器 |
| 自动测量 | `MESCinterface.c` | R/L/磁链/HFI/死区时间自动测量 |
| 高频注入 | `hfi.c` | 零低速无感位置检测 |
| ADC 校准 | `calibrate.c` | ADC 偏置校准终端应用 |
| 通信 | `task_can.c` | CAN 总线遥测与控制 |
| 终端 | `TTerm.c` | 串口终端、变量实时调节 |
| 变量管理 | `TTerm_var.c` | 运行时变量注册与读写 |

---

## 四、核心代码深度分析

### 4.1 硬件初始化与 ADC 增益计算（MESChw_setup.c）

这是整个电机控制的基础——将 ADC 原始读数转换为物理量（安培、伏特）。以下是 `hw_init()` 函数的核心实现：

```cpp
hw_setup_s g_hw_setup;
motor_s motor;

uint32_t ADC_buffer[10];

void hw_init(MESC_motor_typedef *_motor) {
  g_hw_setup.Imax = ABS_MAX_PHASE_CURRENT;
  g_hw_setup.Vmax = ABS_MAX_BUS_VOLTAGE;
  g_hw_setup.Vmin = ABS_MIN_BUS_VOLTAGE;
  g_hw_setup.Rshunt = R_SHUNT;
  g_hw_setup.RVBB = R_VBUS_BOTTOM;
  g_hw_setup.RVBT = R_VBUS_TOP;
  g_hw_setup.OpGain = OPGAIN;
  g_hw_setup.VBGain =
      (3.3f / 4096.0f) * (g_hw_setup.RVBT + g_hw_setup.RVBB) / g_hw_setup.RVBB;
  g_hw_setup.Igain = 3.3f / (g_hw_setup.Rshunt * 4096.0f * g_hw_setup.OpGain * SHUNT_POLARITY);
  g_hw_setup.RawCurrLim =
      g_hw_setup.Imax * g_hw_setup.Rshunt * g_hw_setup.OpGain * (4096.0f / 3.3f)
      + 2048.0f;
  if (g_hw_setup.RawCurrLim > 4000) {
    g_hw_setup.RawCurrLim = 4000;
  }
  g_hw_setup.RawVoltLim =
      (uint16_t)(4096.0f * (g_hw_setup.Vmax / 3.3f) * g_hw_setup.RVBB /
                 (g_hw_setup.RVBT + g_hw_setup.RVBB));
}
```

**逐行深度解析：**

1. **`g_hw_setup.Imax = ABS_MAX_PHASE_CURRENT`**：设置最大相电流限制。这是硬件能测量的上限（ADC 满量程对应的电流）或安全上限，取两者较小值

2. **`g_hw_setup.VBGain`**：母线电压增益计算。公式分解：
   - `3.3f / 4096.0f`：ADC 分辨率，12 位 ADC，参考电压 3.3V，每 LSB = 3.3/4096 ≈ 0.806mV
   - `(RVBT + RVBB) / RVBB`：分压比。例如上臂 100kΩ、下臂 10kΩ，分压比为 11，即实际电压 = ADC 读数 × 11 × 0.806mV

3. **`g_hw_setup.Igain`**：电流增益计算。公式分解：
   - `3.3f / (Rshunt × 4096 × OpGain × SHUNT_POLARITY)`
   - `Rshunt`：分流电阻值（如 1mΩ）
   - `OpGain`：运放增益（如 50 倍）
   - `SHUNT_POLARITY`：极性修正（±1），取决于分流电阻位置（高边/低边）和运放方向
   - 最终：实际电流(A) = (ADC原始值 - 2048) × Igain，其中 2048 是零电流偏置点（ADC 中点）

4. **`g_hw_setup.RawCurrLim`**：计算电流限制对应的 ADC 原始值。公式逻辑：
   - `Imax × Rshunt × OpGain`：最大电流在分流电阻上的压降经运放放大后的电压
   - `× (4096 / 3.3)`：将电压转换为 ADC 计数值
   - `+ 2048`：加上零点偏置（因为电流可正可负，ADC 中点代表零电流）
   - `if > 4000 则限幅`：保留 96 个 LSB 的余量，因为运放可能无法完全轨到轨（rail-to-rail）

5. **`g_hw_setup.RawVoltLim`**：过压保护阈值，直接在 ADC 域计算，无需转换到电压域再做比较

### 4.2 ADC 原始数据读取（MESChw_setup.c）

```cpp
void getRawADC(MESC_motor_typedef *_motor) {
  _motor->Raw.Iu = hadc1.Instance->JDR1;  // U Current
  _motor->Raw.Iv = hadc1.Instance->JDR2;  // V Current
  _motor->Raw.Iw = hadc1.Instance->JDR3;  // W Current
  _motor->Raw.Vbus = hadc1.Instance->JDR4; // DC Link Voltage

  _motor->Raw.ADC_in_ext1 = ADC_buffer[4];  // Throttle
  _motor->Raw.ADC_in_ext2 = ADC_buffer[5];  // Throttle
  _motor->Raw.Motor_T = ADC_buffer[6];       // Motor temperature
}
```

**逐行解析：**

- **`hadc1.Instance->JDR1~JDR4`**：直接读取 ADC 注入通道寄存器。STM32 的 ADC 注入转换（Injected Conversion）与 TIM1 的 PWM 中心对齐模式同步——在 PWM 周期的"过零点"（所有相位占空比相同，无开关噪声）时自动触发，确保电流采样不受开关噪声干扰。这是 FOC 控制的时序关键

- **`ADC_buffer[]`**：通过 DMA 自动填充的常规转换缓冲区。与注入转换不同，常规转换在后台持续运行，用于非时间关键的信号（油门、温度）

- **三电流采样策略**：同时读取 Iu/Iv/Iw 三相电流。根据基尔霍夫电流定律 Iu+Iv+Iw=0，实际上只需两相即可，但三相采样可以做冗余校验和更好的噪声抑制

### 4.3 自动电阻测量（MESCinterface.c）

这是 MESC 最核心的自动校准功能之一。通过施加不同电流并测量电压降来计算电机相电阻：

```cpp
if(measure_res){
  float old_L_D = motor_curr->m.L_D;
  motor_curr->m.R = 0.0001f;  // 0.1mohm, really low
  motor_curr->m.L_D = 0.000001f;  // 1uH, really low
  calculateGains(motor_curr);
  calculateVoltageGain(motor_curr);
  motor_curr->MotorState = MOTOR_STATE_RUN;
  motor_curr->MotorSensorMode = MOTOR_SENSOR_MODE_OPENLOOP;
  motor_curr->FOC.openloop_step = 0;
  
  int a=200;
  float Itop = 0.0f;
  float Ibot = 0.0f;
  float Vtop = 0.0f;
  float Vbot = 0.0f;
  
  // 施加 45% 最大电流
  motor_curr->input_vars.UART_req = 0.45f * motor_curr->m.Imax;
  while(a){
    Ibot = Ibot + motor_curr->FOC.Idq.q;
    Vbot = Vbot + motor_curr->FOC.Vdq.q;
    a--;
    motor_curr->FOC.FOCAngle += 300;
  }
  
  // 施加 55% 最大电流
  a=200;
  motor_curr->input_vars.UART_req = 0.55f * motor_curr->m.Imax;
  while(a){
    Itop = Itop + motor_curr->FOC.Idq.q;
    Vtop = Vtop + motor_curr->FOC.Vdq.q;
    a--;
    motor_curr->FOC.FOCAngle += 300;
  }
  
  // 欧姆定律计算电阻
  motor_curr->m.R = (Vtop - Vbot) / ((Itop - Ibot));
}
```

**逐段深度解析：**

1. **预设置极小值**：`m.R = 0.0001f` 和 `m.L_D = 0.000001f` 设置极小的初始值。这是因为 PI 控制器的增益由 R 和 L 计算（`calculateGains`），极小的 R/L 会导致极高的增益，使控制器对电压变化极其敏感——这在开环测量中反而是好事，因为控制器会快速稳定

2. **开环模式**：`MOTOR_SENSOR_MODE_OPENLOOP` 不依赖位置传感器，直接强制施加电压矢量。`FOCAngle += 300` 每步旋转电角度 300（约 83°），使电机缓慢旋转避免堵转

3. **两点测量法**：施加两个不同的电流（45% 和 55% Imax），各采样 200 次取平均。使用差分计算 `R = ΔV / ΔI` 消除了偏置误差——这是经典的四线测量思路在固件中的实现

4. **为什么不用单点**：单点测量 `R = V/I` 会受 ADC 偏置、死区时间、逆变器管压降等系统性误差影响。两点差分 `(V2-V1)/(I2-I1)` 消除了所有加性偏置，只保留与电流成正比的电阻分量

5. **FOCAngle += 300 的作用**：在开环模式下旋转电角度，确保测量的不是某个特定位置的电阻（齿槽效应可能导致不同位置电阻略有差异），而是平均电阻值

### 4.4 自动电感测量（MESCinterface.c）

电感测量使用了更精巧的高频注入方法：

```cpp
if(measure_ind){
  motor_curr->MotorState = MOTOR_STATE_RUN;
  motor_curr->input_vars.UART_req = 0.25f;
  motor_curr->HFI.Type = HFI_TYPE_SPECIAL;
  motor_curr->HFI.special_injectionVd = 0.2f;
  motor_curr->HFI.special_injectionVq = 0.0f;
  motor_curr->MotorSensorMode = MOTOR_SENSOR_MODE_OPENLOOP;
  motor_curr->FOC.openloop_step = 0;
  motor_curr->FOC.FOCAngle = 0;
  motor_curr->input_vars.UART_dreq = -5.0f;
  
  // 自动调整注入电压直到电流响应足够大
  int a=200;
  while(a){
    if(fabsf(motor_curr->FOC.didq.d) < 5.0f){
      motor_curr->HFI.special_injectionVd *= 1.05f;
      if(motor_curr->HFI.special_injectionVd > (0.5f * motor_curr->Conv.Vbus))
        motor_curr->HFI.special_injectionVd = 0.5f * motor_curr->Conv.Vbus;
    }
    a--;
  }
  
  // 在三个不同 d 轴电流下测量 Ld
  for(b=0; b<3; b++){
    Loffset[b] = 0.0f;
    a=200;
    motor_curr->input_vars.UART_dreq = -motor_curr->m.Imax * 0.25f * (float)b;
    while(a){
      Loffset[b] = Loffset[b] + motor_curr->FOC.didq.d;
      a--;
    }
    // L = V × T_period / ΔI
    Loffset[b] = motor_curr->FOC.pwm_period * motor_curr->HFI.special_injectionVd / Loffset[b];
  }
}
```

**逐段深度解析：**

1. **HFI_TYPE_SPECIAL 模式**：注入一个固定的 d 轴电压（`injectionVd = 0.2f`），不注入 q 轴。这种特殊注入模式专为电感测量设计，不影响电机旋转

2. **自适应电压调整**：`if(fabsf(didq.d) < 5.0f) injectionVd *= 1.05f`——如果电流变化率太小（电感太大导致电流响应弱），自动增加注入电压，步进 5%。上限设为母线电压的一半，防止过压

3. **三点电感测量**：在 0%、25%、50% Imax 的 d 轴电流下分别测量电感。PMSM 电机的电感会随电流变化（磁饱和效应），多点测量可以评估饱和特性

4. **电感计算公式**：`L = V × T_period / ΔI`
   - `V`：注入电压（Vd）
   - `T_period`：PWM 周期（秒）
   - `ΔI`：电流变化率（A）
   - 这是基于电感基本方程 `V = L × (di/dt)` 的离散化形式：`L = V × Δt / ΔI`

5. **`didq.d`**：这是 d 轴电流的微分（导数），由 FOC 核心在每个 PWM 周期计算。它直接反映了注入电压引起的电流变化率

### 4.5 磁链自动测量（MESCinterface.c）

磁链（flux linkage）是 PMSM 电机最关键的参数，决定了反电动势常数和 kV rating：

```cpp
if(measure_kv){
  motor_curr->MotorState = MOTOR_STATE_GET_KV;
  while(motor_curr->MotorState == MOTOR_STATE_GET_KV){
    xSemaphoreGive(port->term_block);
    vTaskDelay(200);
    xQueueSemaphoreTake(port->term_block, portMAX_DELAY);
    ttprintf(".");
  }
  ttprintf("Flux linkage = %f mWb\r\n\r\n",
           (double)(motor_curr->m.flux_linkage * 1000.0f));
}
```

**逐段解析：**

1. **`MOTOR_STATE_GET_KV`**：进入专用测量状态。在这个状态下，FOC 核心会施加电流让电机自由旋转到稳态，然后通过反电动势观测器计算磁链

2. **`xSemaphoreGive/Take`**：释放终端信号量让测量任务运行，然后等待 200ms 后重新获取。这是 FreeRTOS 的协作式等待模式——测量在 FOC 中断中进行（高优先级），CLI 任务需要让出锁

3. **磁链测量原理**：当电机稳态旋转时，`Vq = R × Iq + ω × λ`（q 轴电压方程）。已知 Vq、R、Iq 和电角速度 ω，即可求解磁链 λ。这就是为什么电阻测量必须先于磁链测量

### 4.6 变量注册系统（MESCinterface.c）

MESC 的终端变量系统允许运行时实时读写所有控制参数，无需重新编译：

```cpp
void populate_vars(){
  TERM_addVar(mtr[0].m.Pmax,            0.0f, 50000.0f, "par_p_max",   "Max power",              VAR_ACCESS_RW, NULL,      &TERM_varList);
  TERM_addVar(mtr[0].m.IBatmax,         0.0f, 1000.0f,  "par_ibat_max","Max battery current",    VAR_ACCESS_RW, NULL,      &TERM_varList);
  TERM_addVar(mtr[0].m.direction,       0,    1,        "par_dir",     "Motor direction",        VAR_ACCESS_RW, NULL,      &TERM_varList);
  TERM_addVar(mtr[0].m.pole_pairs,      0,    255,      "par_pp",      "Motor pole pairs",       VAR_ACCESS_RW, NULL,      &TERM_varList);
  TERM_addVar(mtr[0].m.RPMmax,          0,    300000,   "par_rpm_max", "Max RPM",               VAR_ACCESS_RW, NULL,      &TERM_varList);
  TERM_addVar(mtr[0].m.flux_linkage,    0.0f, 100.0f,   "par_flux",    "Flux linkage",          VAR_ACCESS_RW, callback,  &TERM_varList);
  TERM_addVar(mtr[0].FOC.ortega_gain,   1.0f, 100000000.0f, "FOC_ortega_gain", "Ortega gain",    VAR_ACCESS_RW, NULL,      &TERM_varList);
}
```

**逐行解析：**

- **`TERM_addVar`** 参数：`(变量引用, 最小值, 最大值, 名称, 描述, 读写权限, 回调函数, 变量列表句柄)`
- **`VAR_ACCESS_RW`**：读写权限，也有只读模式 `VAR_ACCESS_RO`
- **`callback`**：当变量被修改时调用的回调函数。例如 `par_flux`（磁链）被修改时，`callback` 会自动重新计算 PI 增益和电压增益：

```cpp
void callback(TermVariableDescriptor * var){
  calculateFlux(&mtr[0]);
  calculateGains(&mtr[0]);
  calculateVoltageGain(&mtr[0]);
  MESCinput_Init(&mtr[0]);
}
```

这意味着用户在终端修改磁链值后，所有依赖磁链的控制器参数会自动更新，无需重启。这是极其强大的实时调试能力。

### 4.7 ADC 校准终端应用（calibrate.c）

这是一个基于 FreeRTOS 任务的交互式终端应用，用于 ADC 偏置校准：

```cpp
static uint8_t CMD_main(TERMINAL_HANDLE * handle, uint8_t argCount, char ** args){
    uint8_t currArg = 0;
    uint8_t returnCode = TERM_CMD_EXIT_SUCCESS;
    char ** cpy_args=NULL;
    argCount++;
    if(argCount){
        cpy_args = pvPortMalloc(sizeof(char*)*argCount);
        cpy_args[0] = pvPortMalloc(sizeof(APP_NAME));
        cpy_args[0]=memcpy(cpy_args[0], APP_NAME, sizeof(APP_NAME));
        for(;currArg<argCount-1; currArg++){
            uint16_t len = strlen(args[currArg])+1;
            cpy_args[currArg+1] = pvPortMalloc(len);
            memcpy(cpy_args[currArg+1], args[currArg], len);
        }
    }
    TermProgram * prog = pvPortMalloc(sizeof(TermProgram));
    prog->inputHandler = INPUT_handler;
    prog->args = cpy_args;
    prog->argCount = argCount;
    returnCode = xTaskCreate(TASK_main, APP_NAME, APP_STACK, handle,
                             tskIDLE_PRIORITY + 1, &prog->task)
                 ? TERM_CMD_EXIT_PROC_STARTED : TERM_CMD_EXIT_ERROR;
    if(returnCode == TERM_CMD_EXIT_PROC_STARTED)
        TERM_attachProgramm(handle, prog);
    return returnCode;
}
```

**逐段解析：**

1. **参数深拷贝**：将命令行参数复制到堆内存中。因为终端命令是在临时缓冲区中解析的，如果新任务需要异步访问参数，必须复制到持久化内存

2. **`pvPortMalloc`**：FreeRTOS 的内存分配函数，从堆中分配。`APP_STACK = 512`（字，即 2KB）是任务的栈大小

3. **`xTaskCreate`**：创建 FreeRTOS 任务，优先级为 `tskIDLE_PRIORITY + 1`（略高于空闲任务）。校准不是时间关键任务，不需要高优先级

4. **`TERM_attachProgramm`**：将程序附加到终端，使其可以接收键盘输入。`INPUT_handler` 处理方向键导航和选择

5. **`bargraph` 函数**：终端中绘制柱状图显示 ADC 值，使用 `#` 和 `_` 字符：

```cpp
static void bargraph(TERMINAL_HANDLE * handle, float min, float max, float val){
    char buffer[45];
    memset(buffer,0,45);
    buffer[0]='|';
    float norm = 40.0/max*val;
    for(int i=0; i<40;i++){
        buffer[i+1] = i<norm? '#' : '_';
    }
    buffer[41]='|';
    ttprintf("%s %10.0f ", buffer, (double)val);
}
```

这是一个在纯文本终端中实现可视化反馈的巧妙方法，40 个字符宽的柱状图可以直观显示 ADC 读数的实时变化。

---

## 五、API 使用指南

### 终端命令

MESC 通过 USB CDC 或 UART 提供交互式终端：

```bash
# 连接串口（假设 /dev/ttyACM0）
screen /dev/ttyACM0 115200

# 测量所有电机参数
measure -a

# 仅测量电阻和电感
measure -r

# 测量磁链（flux linkage / kV）
measure -f

# 测量 HFI 阈值
measure -h

# 测量死区时间补偿
measure -d

# 设置开环测量电流
measure -c 2.5

# 设置 HFI 注入电压
measure -v 0.3
```

### 运行时变量调节

```bash
# 查看所有变量
vars

# 设置最大功率
set par_p_max 1500

# 设置电机方向
set par_dir 1

# 设置极对数
set par_pp 7

# 设置最大转速
set par_rpm_max 8000

# 设置磁链（修改后自动重新计算 PI 增益）
set par_flux 0.005

# 设置 Ortega 观测器增益
set FOC_ortega_gain 1000000

# 查看错误标志
error
```

### cURL 通过 CAN 网关控制（如果配置了 CAN-USB 网关）

```bash
# 发送油门指令（CAN ID 0x001）
cansend can0 001#0064  # 100（0x64）= 10% 油门

# 查询状态（CAN ID 0x002）
cansend can0 002#00000000
```

### Python 集成示例

```python
import serial
import time

# 通过 USB CDC 连接
ser = serial.Serial('/dev/ttyACM0', 115200, timeout=1)
time.sleep(2)  # 等待连接稳定

# 自动测量所有电机参数
ser.write(b'measure -a\n')
time.sleep(30)  # 等待测量完成

# 读取输出
while ser.in_waiting:
    line = ser.readline().decode('utf-8', errors='ignore').strip()
    if line:
        print(line)

# 设置参数
ser.write(b'set par_pp 7\n')        # 7 极对
time.sleep(0.1)
ser.write(b'set par_rpm_max 8000\n') # 最大 8000 RPM
time.sleep(0.1)
ser.write(b'set par_dir 0\n')        # 正转
time.sleep(0.1)

# 启动电机
ser.write(b'start\n')
time.sleep(1)

# 设置油门 (0-1)
ser.write(b'throttle 0.3\n')  # 30% 油门

# 实时监控
while True:
    ser.write(b'status\n')
    time.sleep(0.1)
    while ser.in_waiting:
        line = ser.readline().decode('utf-8', errors='ignore').strip()
        if line:
            print(f'[TELEMETRY] {line}')
    time.sleep(0.5)
```

---

## 六、编译与部署

### 环境搭建

| 步骤 | 工具 | 说明 |
|------|------|------|
| IDE | STM32CubeIDE | 推荐，免费 |
| 工具链 | ARM GCC | CubeIDE 自带 |
| 调试器 | ST-Link V2/V3 | 或 J-Link |
| 库 | STM32 HAL | CubeMX 生成 |

### 目标板支持

| 目标板 | MCU | 说明 |
|--------|-----|------|
| MESC_L431RC | STM32L431RC | 低功耗，64MHz |
| MESC_F411CE | STM32F411CE | 高性能，100MHz |
| Axis-Throttle | STM32L432 | 油门手柄控制器 |

### 编译配置

1. **导入工程**
   - STM32CubeIDE → File → Import → Existing Projects
   - 选择 `MESC_L431RC/` 或 `MESC_F411CE/` 目录

2. **硬件参数配置**
   - 编辑 `MESChw_setup.h`，设置分流电阻值、运放增益、母线电压分压比：

   ```c
   #define R_SHUNT          0.001f   // 1mΩ 分流电阻
   #define OPGAIN           50.0f    // 运放增益 50 倍
   #define R_VBUS_TOP       100000.0f  // 母线电压上臂 100kΩ
   #define R_VBUS_BOTTOM    10000.0f   // 母线电压下臂 10kΩ
   #define ABS_MAX_PHASE_CURRENT  40.0f  // 最大相电流 40A
   #define ABS_MAX_BUS_VOLTAGE    60.0f  // 最大母线电压 60V
   #define ABS_MIN_BUS_VOLTAGE    10.0f  // 最小母线电压 10V
   ```

3. **编译与烧录**
   - Build → 编译生成 .elf/.hex
   - Debug/Run → 通过 ST-Link 烧录

### 电机调试流程

| 步骤 | 命令 | 说明 |
|------|------|------|
| 1 | `calibrate` | ADC 偏置校准（电机断电） |
| 2 | `measure -a` | 自动测量 R/L/磁链 |
| 3 | `set par_pp 7` | 设置极对数 |
| 4 | `set par_dir 0` | 设置方向 |
| 5 | `start` | 启动电机 |
| 6 | `throttle 0.1` | 缓慢加速测试 |

---

## 七、项目亮点与适用场景

### 技术亮点

1. **全自动电机参数辨识**：无需数据手册，固件自动测量 R/L/磁链/HFI 阈值/死区时间。两点差分法消除偏置误差，多点电感测量评估磁饱和特性
2. **多观测器架构**：支持滑模观测器(SMO)、磁链观测器(Ortega)、HFI 高频注入，覆盖从零速到高速的全速域无感控制
3. **运行时变量系统**：TTerm 变量管理系统允许实时调节 50+ 控制参数，修改磁链等关键参数后自动重算 PI 增益
4. **弱磁与 MTPA**：支持 Field Weakening 扩展恒功率区，MTPA 策略最大化转矩效率
5. **跨平台设计**：核心 FOC 库与硬件抽象层分离，同一套代码支持 STM32L4/F4 等多个系列
6. **终端可视化**：纯文本柱状图和 VT100 控制码实现实时数据可视化，无需上位机软件

### 适用场景

| 场景 | 说明 |
|------|------|
| 电动车辆控制器 | 电动滑板/自行车/无人机 ESC |
| 伺服驱动器 | 工业位置/速度伺服 |
| FOC 学习参考 | 完整的开源 FOC 实现范例 |
| 电机参数辨识 | 自动测量算法可移植 |
| RTOS 电机控制 | FreeRTOS + FOC 集成参考 |
| CAN 总线节点 | 多电机协调控制 |

---

## 八、总结

MESC_Firmware 是一个在开源 FOC 电机控制领域极具深度的项目。它的核心价值不在于"又造了一个 ESC"，而在于其全自动参数辨识系统——通过精巧的两点差分电阻测量、自适应高频注入电感测量、稳态反电动势磁链测量，实现了"接上未知电机就能自动调参并运行"的能力。配合 TTerm 运行时变量系统和 FreeRTOS 多任务架构，它在电机控制的学习、原型开发和实际应用中都极具参考价值。BSD-3-Clause 许可证也允许商业使用，对于需要电机控制的 产品开发团队来说是一个值得深入研究的代码库。

---

📝 作者：蔡浩宇（jun-chy） / 📅 日期：2026-07-04 / 🔗 项目地址：[https://github.com/davidmolony/MESC_Firmware](https://github.com/davidmolony/MESC_Firmware)
