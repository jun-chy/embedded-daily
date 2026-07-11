# STM32 IMU 扩展卡尔曼滤波器（EKF）深度解析 — 6-DOF 姿态估计从数学到固件

> 📅 日期：2026-07-11 | 🏷️ 分类：STM32 / 传感器融合 | ⭐ Stars：56 | 作者：蔡浩宇（jun-chy）

---

## 项目信息

| 属性 | 值 |
|------|------|
| **项目地址** | [DjVul/Extended-Kalman-Filter---STM32](https://github.com/DjVul/Extended-Kalman-Filter---STM32) |
| **作者** | DjVul |
| **许可证** | Apache License 2.0 |
| **最近更新** | 2026-07-10 |
| **创建时间** | 2026-07-05 |
| **主要语言** | C (92.3%)、Python (7.7%) |
| **Forks** | 10 |
| **目标平台** | STM32F3xx（可移植至任意 MCU） |

---

## 一、项目简介

本项目在 STM32 微控制器上实现了一个完整的 **6-DOF IMU 扩展卡尔曼滤波器（EKF）**，通过融合三轴陀螺仪和三轴加速度计数据，实时估计 Roll（横滚角）和 Pitch（俯仰角）。项目不仅包含嵌入式 C 固件，还配套了 Python 数据生成与可视化工具，形成从仿真到真机的完整验证闭环。

与简单的互补滤波器不同，EKF 通过 **雅可比矩阵线性化** 非线性测量模型，在数学最优的意义上融合传感器数据。项目作者将整个算法逐行注释，使未实现过 EKF 的开发者也能理解每一步的数学含义。

### 核心特性一览

| 特性 | 描述 |
|------|------|
| 6 维状态向量 | `[roll, pitch, yaw, bias_x, bias_y, bias_z]`，同时估计姿态角与陀螺仪零偏 |
| EKF 双阶段 | 预测阶段（陀螺仪积分）+ 校正阶段（加速度计重力向量校正） |
| 雅可比矩阵 | 手动推导 3×6 测量雅可比，线性化 sin/cos 非线性模型 |
| 平台无关 | 核心 EKF 与矩阵运算库零平台依赖，可移植至 Arduino/ESP32 等 |
| 3×3 矩阵求逆 | 使用伴随矩阵法直接求逆，避免迭代式数值方法 |
| UART 实时通信 | 12 字节二进制协议，100 Hz 采样率，中断驱动收发 |
| Python 仿真链 | NumPy 数据生成 + PySerial UART 通信 + Matplotlib 可视化 |
| 陀螺仪零偏补偿 | 在线估计并扣除陀螺仪三轴零偏漂移 |

---

## 二、硬件架构

### 系统组成图

```
┌─────────────────────────────────────────────────────────────┐
│                      STM32F303RE (Nucleo)                    │
│                                                             │
│  ┌──────────┐    ┌──────────────────┐    ┌───────────────┐  │
│  │ UART RX  │───▶│  EKF 处理引擎     │───▶│  UART TX     │  │
│  │ (PA3)    │    │                  │    │  (PA2)       │  │
│  │ 12 bytes │    │  1. 预测(陀螺仪)  │    │  12 bytes    │  │
│  │ 6×int16  │    │  2. 校正(加速度计)│    │  3×float32   │  │
│  └──────────┘    │  3. 状态输出     │    └───────────────┘  │
│       ▲          └──────────────────┘           │           │
│       │                  ▲                      │           │
│       │                  │                      │           │
│  ┌────┴──────┐    ┌─────┴──────┐         ┌─────┴─────┐    │
│  │  外部 IMU  │    │  矩阵运算库 │         │  PC 端    │    │
│  │  传感器    │    │  matrix_utils│        │  Python   │    │
│  │  MPU6050  │    │  (add/mul/  │         │  可视化   │    │
│  │  /MPU9250 │    │   inv/...)  │         │           │    │
│  └───────────┘    └────────────┘         └───────────┘    │
│                                                             │
│  ┌─────────┐  ┌──────────┐  ┌──────────────────────┐       │
│  │ HSI 8MHz│─▶│ PLL ×16  │─▶│ SYSCLK = 64 MHz      │       │
│  └─────────┘  └──────────┘  └──────────────────────┘       │
│                                                             │
│  ┌──────────────────────────────────────┐                  │
│  │ LED (PB5/LD2)  按钮 (PC13/B1)        │                  │
│  └──────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘

         UART 协议 (115200 baud, 8N1)
         ┌─────────────────────────────────────────┐
  PC ──▶ │ [gx_lo][gx_hi][gy_lo][gy_hi][gz_lo]... │ ──▶ STM32
  RX ◀── │ [roll_4B][pitch_4B][yaw_4B]            │ ◀── TX
         └─────────────────────────────────────────┘
         12 bytes (6×int16, little-endian)         12 bytes (3×float32)
```

### 引脚配置表

| 外设 | 引脚 | 功能 | 说明 |
|------|------|------|------|
| USART2_TX | PA2 | UART 发送 | EKF 滤波结果输出（3×float32） |
| USART2_RX | PA3 | UART 接收 | IMU 原始数据输入（6×int16） |
| LD2 | PB5 | LED 指示 | Nucleo 板载 LED |
| B1 | PC13 | 按钮输入 | Nucleo 板载按钮（下降沿中断） |
| TMS | PA13 | SWDIO | 调试接口 |
| TCK | PA14 | SWCLK | 调试接口 |
| SWO | PB3 | SWO 跟踪 | 调试跟踪接口 |

### 硬件清单

| 组件 | 型号 | 数量 | 用途 |
|------|------|------|------|
| MCU 开发板 | STM32 Nucleo-F303RE | 1 | 主控制器，Cortex-M4F @ 64 MHz |
| IMU 传感器 | MPU6050 / MPU9250 / ICM-42688 | 1 | 6 轴陀螺仪+加速度计（或外部模拟数据源） |
| USB-TTL 模块 | CP2102 / CH340 | 1 | PC 与 STM32 之间 UART 通信（如非 Nucleo 板载） |
| 连接线 | 杜邦线 | 若干 | 传感器与 MCU 互连 |

> **说明**：项目目前使用 Python 脚本通过 UART 向 STM32 发送合成 IMU 数据，验证 EKF 算法正确性。实际使用时可将 IMU 传感器数据直接通过 I2C/SPI 读入，替换 UART 接收逻辑。

---

## 三、固件架构

### 文件结构

```
Extended-Kalman-Filter---STM32/
├── Core/
│   ├── Inc/
│   │   ├── main.h                          # STM32 HAL 引脚定义与硬件配置
│   │   ├── orientation_ekf_rp.h             # EKF 接口声明与数据类型定义
│   │   ├── orientation_ekf_rp_config.h      # EKF 参数配置（Q/R/P 维度）
│   │   ├── matrix_utils.h                   # 矩阵运算函数声明
│   │   └── linear_algebra.h                # 线性代数模块入口（预留四元数扩展）
│   └── Src/
│       ├── main.c                          # STM32 主程序：UART 收发 + EKF 调用
│       ├── orientation_ekf_rp.c            # EKF 核心实现：预测+校正+输出
│       └── matrix_utils.c                  # 矩阵运算实现：加减乘转置求逆
├── Python/
│   ├── IMU_GyroAcc_Data_Generator.py       # IMU 数据生成器（真值+噪声+零偏）
│   ├── imu_uart_simulator.py               # UART 仿真器：发送数据+接收结果+绘图
│   └── imu_data.h                          # 预生成 IMU 数据头文件（1000 样本）
├── KalmanFilterDiagram.png                 # EKF 原理框图
├── FilteringFullSignal.png                # 完整滤波结果对比图
├── Last2SecondOfFIltering.png              # 最后 2 秒放大对比图
├── LICENSE                                 # Apache-2.0
└── README.md
```

### 模块职责表

| 模块文件 | 行数 | 职责 |
|---------|------|------|
| `orientation_ekf_rp.c` | ~250 行 | EKF 核心：状态初始化、预测（陀螺仪积分）、校正（加速度计重力向量）、角度输出 |
| `matrix_utils.c` | ~90 行 | 矩阵运算库：加减、乘法、转置、标量乘、拷贝、3×3 伴随矩阵求逆 |
| `main.c` | ~200 行 | STM32 HAL 初始化、UART 中断收发、数据解包与 EKF 调用 |
| `orientation_ekf_rp_config.h` | ~30 行 | 滤波器参数：状态维度、噪声协方差 Q/R、初始协方差 P |
| `imu_uart_simulator.py` | ~150 行 | PC 端：生成合成 IMU 数据、UART 通信、结果可视化 |
| `IMU_GyroAcc_Data_Generator.py` | ~130 行 | 独立 IMU 数据生成与可视化工具 |

---

## 四、核心代码深度分析

### 4.1 EKF 参数配置（`orientation_ekf_rp_config.h`）

```c
// ===================== DIMENZIJE =====================
#define EKF_STATE_DIM     6       // 状态向量维度：6
#define EKF_MEAS_ACCEL    3       // 测量向量维度：3

// ===================== INDEKSI =====================
#define IDX_ROLL    0             // 状态向量索引：Roll
#define IDX_PITCH   1             //                  Pitch
#define IDX_YAW     2             //                  Yaw
#define IDX_BIAS_X  3             //                  陀螺仪 X 轴零偏
#define IDX_BIAS_Y  4             //                  陀螺仪 Y 轴零偏
#define IDX_BIAS_Z  5             //                  陀螺仪 Z 轴零偏

// ===================== MATRICE ŠUMA (KONSTANTE) =====================
#define Q_ANGLE     0.0001f       // 角度过程噪声协方差
#define Q_BIAS      0.00001f      // 零偏过程噪声协方差
#define R_ACCEL     1.0f          // 加速度计测量噪声协方差

// ===================== INICIJALIZACIJA =====================
#define P_INIT      0.1f         // 初始估计协方差
```

**逐行解析：**

- `EKF_STATE_DIM = 6`：定义状态向量维度为 6，包含 3 个姿态角和 3 个陀螺仪零偏。这是典型的 6-DOF EKF 状态表示。
- `EKF_MEAS_ACCEL = 3`：加速度计提供 3 轴测量（ax, ay, az），测量向量维度为 3。
- `IDX_*` 宏：定义状态向量各分量在数组中的索引，使代码可读性大幅提升，避免魔法数字。
- `Q_ANGLE = 0.0001f`：角度状态的过程噪声协方差。值越小表示对模型越信任；0.0001 是较小的值，说明作者认为陀螺仪积分模型较为可靠。
- `Q_BIAS = 0.00001f`：零偏状态的过程噪声。比角度噪声小一个数量级，因为陀螺仪零偏变化缓慢，模型假设其近似恒定。
- `R_ACCEL = 1.0f`：加速度计测量噪声协方差。值为 1.0 相对较大，反映加速度计容易受振动和运动加速度干扰。
- `P_INIT = 0.1f`：初始状态估计协方差。设为中等值，表示初始姿态估计有一定不确定性但不至于完全未知。

**参数调谐含义**：Q/R 的比值决定了滤波器对陀螺仪 vs 加速度计的信任程度。本项目 Q_ANGLE（0.0001）远小于 R_ACCEL（1.0），意味着滤波器在短期内高度信任陀螺仪积分，仅在长期用加速度计缓慢校正漂移。

---

### 4.2 EKF 状态与接口定义（`orientation_ekf_rp.h`）

```c
typedef struct {
    float roll;          // radijani
    float pitch;         // radijani
    float yaw;           // radijani
} ekf_angles_t;

typedef struct {
    float roll;          // radijani
    float pitch;         // radijani
    float yaw;           // radijani
    float bias_x;        // rad/s
    float bias_y;        // rad/s
    float bias_z;        // rad/s
} ekf_state_t;

#define DEG_TO_RAD(deg) ((deg) * 3.141592653589793f / 180.0f)
#define RAD_TO_DEG(rad) ((rad) * 180.0f / 3.141592653589793f)

void ekf_init(void);
void ekf_predict(const float dt, const float gx, const float gy, const float gz);
void ekf_update_accel(const float ax, const float ay, const float az);
void ekf_get_angles(float *roll, float *pitch, float *yaw);
void ekf_get_radians(float *roll, float *pitch, float *yaw);
void ekf_get_state(ekf_state_t *state);
```

**逐段解析：**

- `ekf_angles_t`：仅包含三个姿态角，用于简单场景的输出获取。
- `ekf_state_t`：完整状态结构体，包含姿态角和陀螺仪零偏，用于需要监控零偏估计值的场景。
- `DEG_TO_RAD / RAD_TO_DEG`：角度-弧度转换宏，使用高精度 π 值（3.141592653589793f），确保浮点精度。
- `ekf_init()`：初始化滤波器，设置 Q/P/R 矩阵并清零状态。
- `ekf_predict()`：预测阶段入口，输入时间步长 dt 和三轴角速度（含零偏的原始数据）。
- `ekf_update_accel()`：校正阶段入口，输入三轴加速度测量值。
- `ekf_get_angles()` / `ekf_get_radians()`：分别以度和弧度获取姿态角。
- `ekf_get_state()`：获取完整 6 维状态，用于调试零偏估计。

接口设计的关键点：**预测和校正分离**。调用者可以在只有陀螺仪数据时仅执行预测，在有加速度计数据时再执行校正，灵活适应不同采样率的传感器。

---

### 4.3 EKF 核心实现 — 初始化（`orientation_ekf_rp.c`）

```c
static float x[EKF_STATE_DIM];                         // 状态向量
static float P[EKF_STATE_DIM][EKF_STATE_DIM];          // 协方差矩阵
static float Q[EKF_STATE_DIM][EKF_STATE_DIM];          // 过程噪声协方差
static float R_accel[EKF_MEAS_ACCEL][EKF_MEAS_ACCEL];  // 测量噪声协方差

static void init_QPR(void)
{
    memset(Q, 0, sizeof(Q));
    Q[IDX_ROLL][IDX_ROLL]   = Q_ANGLE;
    Q[IDX_PITCH][IDX_PITCH] = Q_ANGLE;
    Q[IDX_YAW][IDX_YAW]     = Q_ANGLE;
    Q[IDX_BIAS_X][IDX_BIAS_X] = Q_BIAS;
    Q[IDX_BIAS_Y][IDX_BIAS_Y] = Q_BIAS;
    Q[IDX_BIAS_Z][IDX_BIAS_Z] = Q_BIAS;

    memset(P, 0, sizeof(P));
    for(uint8_t i = 0; i < EKF_STATE_DIM; i++)
        P[i][i] = P_INIT;

    memset(R_accel, 0, sizeof(R_accel));
    for(uint8_t i = 0; i < EKF_MEAS_ACCEL; i++)
        R_accel[i][i] = R_ACCEL;
}

void ekf_init(void)
{
    init_QPR();
    memset(x, 0, sizeof(x));
}
```

**逐行解析：**

- **静态全局变量**：`x`（状态）、`P`（协方差）、`Q`（过程噪声）、`R_accel`（测量噪声）均为 `static` 全局变量，确保模块封装性——外部无法直接访问滤波器内部状态，只能通过 API 函数获取结果。
- `init_QPR()`：
  - **Q 矩阵**：6×6 对角矩阵，角度分量对角元为 `Q_ANGLE`（0.0001），零偏分量对角元为 `Q_BIAS`（0.00001）。非对角元为零，假设各状态噪声独立。
  - **P 矩阵**：6×6 对角矩阵，所有对角元初始化为 `P_INIT`（0.1），表示初始时各状态估计的不确定性相同。
  - **R 矩阵**：3×3 对角矩阵，所有对角元为 `R_ACCEL`（1.0），假设三轴加速度计噪声独立且方差相同。
- `ekf_init()`：先初始化 Q/P/R，再将状态向量 `x` 清零——假设系统初始姿态为水平（roll=pitch=yaw=0），陀螺仪零偏初始为零。

> **设计要点**：所有矩阵使用二维静态数组而非动态分配，在嵌入式环境中避免了堆内存碎片问题，且编译期即可确定内存布局。

---

### 4.4 EKF 核心实现 — 预测阶段（`orientation_ekf_rp.c`）

```c
void ekf_predict(const float dt, const float gx, const float gy, const float gz)
{
    // 状态外推：x[n+1|n] = f(x[n|n])
    x[IDX_ROLL]  += (gx - x[IDX_BIAS_X]) * dt;
    x[IDX_PITCH] += (gy - x[IDX_BIAS_Y]) * dt;
    x[IDX_YAW]   += (gz - x[IDX_BIAS_Z]) * dt;

    // 状态转移矩阵 F (6x6)
    float F[EKF_STATE_DIM][EKF_STATE_DIM] = {
        {1, 0, 0, -dt, 0, 0},
        {0, 1, 0, 0, -dt, 0},
        {0, 0, 1, 0, 0, -dt},
        {0, 0, 0, 1, 0, 0},
        {0, 0, 0, 0, 1, 0},
        {0, 0, 0, 0, 0, 1}
    };

    // 协方差外推：P[n+1|n] = F * P[n|n] * F^T + Q
    float FP[EKF_STATE_DIM][EKF_STATE_DIM];     // F * P
    float FPFt[EKF_STATE_DIM][EKF_STATE_DIM];   // F * P * F^T

    // FP = P * F^T（利用矩阵乘转置优化）
    matrix_mul_transpose((float*)P, (float*)F, (float*)FP,
                         EKF_STATE_DIM, EKF_STATE_DIM, EKF_STATE_DIM);
    // FPFt = F * FP = F * P * F^T
    matrix_mul((float*)F, (float*)FP, (float*)FPFt,
               EKF_STATE_DIM, EKF_STATE_DIM, EKF_STATE_DIM);
    // P = FPFt + Q
    matrix_add((float*)FPFt, (float*)Q, (float*)P,
               EKF_STATE_DIM, EKF_STATE_DIM);
}
```

**逐行深度解析：**

**状态外推部分：**
- `x[IDX_ROLL] += (gx - x[IDX_BIAS_X]) * dt;`：Roll 角的预测。将陀螺仪 X 轴角速度 `gx` 减去估计的零偏 `x[IDX_BIAS_X]`，再乘以时间步长 `dt`，得到角度增量并累加到当前 Roll 估计。这是欧拉角积分的最简形式——假设在 dt 时间内角速度恒定。
- Pitch 和 Yaw 的预测同理，分别使用 gy/gz 和对应零偏。
- **零偏的作用**：陀螺仪静止时输出不为零（零偏漂移），EKF 将零偏作为状态变量在线估计并扣除，这是比简单互补滤波器的核心优势。

**状态转移矩阵 F：**
- F 是 6×6 矩阵，描述状态如何从一个时刻转移到下一时刻。
- 左上 3×3 子矩阵为单位阵——角度状态直接保留。
- 右上 3×3 子矩阵对角元为 `-dt`——表示零偏通过 `dt` 影响角度预测（注意负号，因为零偏被减去）。
- 右下 3×3 子矩阵为单位阵——零偏假设恒定（随机游走模型）。
- 其余元素为零——各轴之间无耦合。

**协方差外推部分：**
- 公式 `P = F * P * F^T + Q` 是标准 EKF 协方差预测方程。
- `matrix_mul_transpose(P, F, FP, ...)`：计算 `P * F^T`，直接利用转置乘法优化避免显式构造 F^T。
- `matrix_mul(F, FP, FPFt, ...)`：计算 `F * (P * F^T)`。
- `matrix_add(FPFt, Q, P, ...)`：加上过程噪声 Q，得到预测协方差。
- **物理含义**：协方差矩阵 P 描述状态估计的不确定性。预测阶段不确定性增大——因为陀螺仪积分会累积误差，Q 矩阵量化了这一不确定性增长。

> **性能考量**：6×6 矩阵乘法在 Cortex-M4F 上约需 216 次乘法+180 次加法。在 64 MHz 主频下约 5-10 微秒，远小于 10 ms（100 Hz）的采样间隔。

---

### 4.5 EKF 核心实现 — 校正阶段（`orientation_ekf_rp.c`）

这是整个 EKF 最复杂也最关键的部分。校正阶段使用加速度计测量来修正陀螺仪积分累积的漂移。

```c
void ekf_update_accel(const float ax, const float ay, const float az)
{
    // ========== 1. 计算期望测量 h(x) ==========
    float roll = x[IDX_ROLL];
    float pitch = x[IDX_PITCH];
    float g = 9.81f;
    float h[EKF_MEAS_ACCEL];
    h[0] = -g * sinf(pitch);                 // 期望 ax
    h[1] =  g * cosf(pitch) * sinf(roll);    // 期望 ay
    h[2] =  g * cosf(pitch) * cosf(roll);    // 期望 az

    // ========== 2. 计算测量残差 y = z - h(x) ==========
    float y[EKF_MEAS_ACCEL];
    y[0] = ax - h[0];
    y[1] = ay - h[1];
    y[2] = az - h[2];
```

**解析 — 测量模型 h(x)：**
- `h[0] = -g * sinf(pitch)`：当设备有俯仰角 pitch 时，重力在 X 轴的分量。负号因为 pitch 正方向定义。
- `h[1] = g * cosf(pitch) * sinf(roll)`：重力在 Y 轴的分量，同时受 pitch 和 roll 影响。
- `h[2] = g * cosf(pitch) * cosf(roll)`：重力在 Z 轴的分量。
- 这是 **非线性函数**——包含 sin/cos 运算，正是需要 EKF（而非标准卡尔曼滤波）的原因。
- **残差 y**：实际加速度计读数减去期望值。如果滤波器估计准确，y 应接近零。

```c
    // ========== 3. 计算测量雅可比矩阵 H (3x6) ==========
    float H[EKF_MEAS_ACCEL][EKF_STATE_DIM] = {0};
    // 第一行：dh_ax/d(pitch) = -g * cos(pitch)
    H[0][IDX_PITCH] = -g * cosf(pitch);
    // 第二行：dh_ay/d(roll) = g * cos(pitch) * cos(roll)
    //         dh_ay/d(pitch) = -g * sin(pitch) * sin(roll)
    H[1][IDX_ROLL]  =  g * cosf(pitch) * cosf(roll);
    H[1][IDX_PITCH] = -g * sinf(pitch) * sinf(roll);
    // 第三行：dh_az/d(roll) = -g * cos(pitch) * sin(roll)
    //         dh_az/d(pitch) = -g * sin(pitch) * cos(roll)
    H[2][IDX_ROLL]  = -g * cosf(pitch) * sinf(roll);
    H[2][IDX_PITCH] = -g * sinf(pitch) * cosf(roll);
```

**解析 — 雅可比矩阵 H：**
- H 是 h(x) 对状态向量 x 的偏导数矩阵，维度 3×6。
- H 的第一行（ax 对状态的偏导）：只有 `H[0][IDX_PITCH]` 非零，因为 `h[0] = -g*sin(pitch)` 仅依赖于 pitch。偏导为 `-g*cos(pitch)`。
- H 的第二行（ay 对状态的偏导）：`h[1] = g*cos(pitch)*sin(roll)` 同时依赖 roll 和 pitch。对 roll 求偏导得 `g*cos(pitch)*cos(roll)`，对 pitch 求偏导得 `-g*sin(pitch)*sin(roll)`。
- H 的第三行（az 对状态的偏导）：`h[2] = g*cos(pitch)*cos(roll)`。对 roll 求偏导得 `-g*cos(pitch)*sin(roll)`，对 pitch 求偏导得 `-g*sin(pitch)*cos(roll)`。
- **注意**：H 中所有对 Yaw 和零偏的偏导为零，因为加速度计测量不直接包含 Yaw 信息（Yaw 需要磁力计），且零偏不直接影响加速度计读数。

```c
    // ========== 4. 计算残差协方差 S = H * P * H^T + R ==========
    float HP[EKF_MEAS_ACCEL][EKF_STATE_DIM];       // H * P (3x6)
    float HPHt[EKF_MEAS_ACCEL][EKF_MEAS_ACCEL];    // H * P * H^T (3x3)
    float S[EKF_MEAS_ACCEL][EKF_MEAS_ACCEL];       // S = HPHt + R (3x3)

    matrix_mul((float*)H, (float*)P, (float*)HP,
               EKF_MEAS_ACCEL, EKF_STATE_DIM, EKF_STATE_DIM);
    matrix_mul_transpose((float*)HP, (float*)H, (float*)HPHt,
                         EKF_MEAS_ACCEL, EKF_STATE_DIM, EKF_MEAS_ACCEL);
    matrix_add((float*)HPHt, (float*)R_accel, (float*)S,
               EKF_MEAS_ACCEL, EKF_MEAS_ACCEL);
```

**解析 — 残差协方差 S：**
- S 描述测量残差的不确定性。如果 S 大，说明残差可能由噪声引起，校正应保守；如果 S 小，说明残差可能反映真实状态偏差，校正应积极。
- `HP = H * P`：将状态不确定性投影到测量空间。3×6 乘 6×6 得 3×6。
- `HPHt = HP * H^T`：3×6 乘 6×3 得 3×3，得到测量空间中的不确定性。
- `S = HPHt + R`：加上测量噪声 R。R 大（加速度计噪声大）→ S 大 → 校正保守。

```c
    // ========== 5. 求 S 的逆 ==========
    float S_inv[EKF_MEAS_ACCEL][EKF_MEAS_ACCEL];
    if (!matrix_inv_3x3(S, S_inv)) {
        return;  // S 奇异，跳过本次校正
    }
```

**解析 — S 矩阵求逆：**
- 使用 `matrix_inv_3x3()` 通过伴随矩阵法直接求逆，避免了迭代式数值方法（如 Gauss-Jordan）的开销。
- **奇异性检查**：如果行列式接近零（`|det| < 1e-10`），说明 S 不可逆，直接跳过本次校正。这是一种保护措施——避免数值不稳定导致滤波器发散。

```c
    // ========== 6. 计算卡尔曼增益 K = P * H^T * S^-1 ==========
    float Ht[EKF_STATE_DIM][EKF_MEAS_ACCEL];       // H^T (6x3)
    float PHt[EKF_STATE_DIM][EKF_MEAS_ACCEL];      // P * H^T (6x3)
    float K[EKF_STATE_DIM][EKF_MEAS_ACCEL];        // Kalman gain (6x3)

    matrix_transpose((float*)H, (float*)Ht, EKF_MEAS_ACCEL, EKF_STATE_DIM);
    matrix_mul((float*)P, (float*)Ht, (float*)PHt,
               EKF_STATE_DIM, EKF_STATE_DIM, EKF_MEAS_ACCEL);
    matrix_mul((float*)PHt, (float*)S_inv, (float*)K,
               EKF_STATE_DIM, EKF_MEAS_ACCEL, EKF_MEAS_ACCEL);
```

**解析 — 卡尔曼增益 K：**
- K 是 6×3 矩阵，决定测量残差对状态修正的权重。
- `K = P * H^T * S^-1`：P 大（状态不确定性大）→ K 大（更信任测量）；S 大（测量噪声大）→ K 小（更信任模型预测）。
- **物理含义**：K 的每个元素 `K[i][j]` 表示第 j 个测量残差对第 i 个状态修正的贡献权重。

```c
    // ========== 7. 状态校正 x = x + K * y ==========
    float Ky[EKF_STATE_DIM];  // K * y (6x1)
    matrix_mul((float*)K, y, Ky, EKF_STATE_DIM, EKF_MEAS_ACCEL, 1);
    for (uint8_t i = 0; i < EKF_STATE_DIM; i++) {
        x[i] += Ky[i];
    }
```

**解析 — 状态更新：**
- `Ky = K * y`：6×3 乘 3×1 得 6×1，即各状态分量的修正量。
- `x[i] += Ky[i]`：将修正量加到状态向量。正值修正表示测量值大于期望值，负值修正表示相反。
- **零偏的隐式校正**：虽然 H 矩阵中零偏列全为零，但由于 P 矩阵中角度与零偏之间存在协方差（通过预测阶段的 F 矩阵引入），K 矩阵中零偏行可能非零。这使得加速度计测量不仅能校正角度，还能间接校正陀螺仪零偏——这是 EKF 相对于互补滤波器的核心优势。

```c
    // ========== 8. 协方差更新 P = (I - K*H) * P * (I - K*H)^T + K*R*K^T ==========
    float KH[EKF_STATE_DIM][EKF_STATE_DIM];       // K * H (6x6)
    float I_KH[EKF_STATE_DIM][EKF_STATE_DIM];     // I - K*H (6x6)
    float T[EKF_STATE_DIM][EKF_STATE_DIM];        // 临时矩阵
    float I[EKF_STATE_DIM][EKF_STATE_DIM];        // 单位矩阵

    matrix_mul((float*)K, (float*)H, (float*)KH,
               EKF_STATE_DIM, EKF_MEAS_ACCEL, EKF_STATE_DIM);

    memset(I, 0, sizeof(I));
    for(uint8_t i = 0; i < EKF_STATE_DIM; i++)
        I[i][i] = 1;

    matrix_sub(I, KH, I_KH, EKF_STATE_DIM, EKF_STATE_DIM);

    // T = I_KH * P
    matrix_mul((float*)I_KH, (float*)P, (float*)T,
               EKF_STATE_DIM, EKF_STATE_DIM, EKF_STATE_DIM);
    // P = T * I_KH^T
    matrix_mul_transpose((float*)T, (float*)I_KH, (float*)P,
                         EKF_STATE_DIM, EKF_STATE_DIM, EKF_STATE_DIM);
    // P = P + K * R * K^T
    float KR[EKF_STATE_DIM][EKF_MEAS_ACCEL];      // K * R (6x3)
    float KRKt[EKF_STATE_DIM][EKF_STATE_DIM];     // K * R * K^T (6x6)
    matrix_mul((float*)K, (float*)R_accel, (float*)KR,
               EKF_STATE_DIM, EKF_MEAS_ACCEL, EKF_MEAS_ACCEL);
    matrix_mul_transpose((float*)KR, (float*)K, (float*)KRKt,
                         EKF_STATE_DIM, EKF_MEAS_ACCEL, EKF_STATE_DIM);
    matrix_add((float*)P, (float*)KRKt, (float*)P,
               EKF_STATE_DIM, EKF_STATE_DIM);
}
```

**解析 — Joseph 形式协方差更新：**
- 作者使用了 **Joseph 形式** `P = (I-KH)*P*(I-KH)^T + K*R*K^T` 而非简化的 `P = (I-KH)*P`。
- Joseph 形式的优势：**保证 P 的对称正定性**，避免数值漂移导致滤波器发散。简化的 `P=(I-KH)*P` 在有限精度运算中可能逐渐失去对称性。
- `KH = K * H`：6×3 乘 3×6 得 6×6。
- `I_KH = I - KH`：单位阵减去 KH，得到"修正矩阵"。
- `T = I_KH * P` 然后 `P = T * I_KH^T`：两步完成 `(I-KH)*P*(I-KH)^T`。
- `KRKt = K * R * K^T`：测量噪声通过卡尔曼增益引入的附加不确定性。
- 最终 `P = (I-KH)*P*(I-KH)^T + K*R*K^T`：校正后的不确定性——总是小于校正前，因为测量提供了新信息。

> **工程价值**：Joseph 形式虽然多一次矩阵乘法，但在长时间运行的嵌入式系统中能有效防止协方差矩阵退化。对于姿态估计这类需要持续运行的滤波器，这一选择体现了作者的工程严谨性。

---

### 4.6 矩阵运算库（`matrix_utils.c`）

```c
// C = A * B  (m x n) * (n x p) = (m x p)
void matrix_mul(const float *A, const float *B, float *C,
                uint8_t m, uint8_t n, uint8_t p)
{
    for (uint8_t i = 0; i < m; i++) {
        for (uint8_t j = 0; j < p; j++) {
            C[i * p + j] = 0.0f;
            for (uint8_t k = 0; k < n; k++) {
                C[i * p + j] += A[i * n + k] * B[k * p + j];
            }
        }
    }
}

// C = A * B^T  (m x n) * (p x n)^T = (m x p)
void matrix_mul_transpose(const float *A, const float *B, float *C,
                          uint8_t m, uint8_t n, uint8_t p)
{
    for (uint8_t i = 0; i < m; i++) {
        for (uint8_t j = 0; j < p; j++) {
            C[i * p + j] = 0.0f;
            for (uint8_t k = 0; k < n; k++) {
                C[i * p + j] += A[i * n + k] * B[j * n + k];  // B^T 即 B[j][k]
            }
        }
    }
}
```

**逐行解析：**

- **一维数组存储**：所有矩阵以一维 `float` 数组存储，通过 `i * n + j` 索引访问。这是嵌入式 C 中常见的做法——避免了二维数组作为函数参数时的尺寸固定问题，使矩阵运算函数完全通用。
- `matrix_mul()`：标准三重循环矩阵乘法。外层遍历结果的行和列，内层计算点积。时间复杂度 O(m×n×p)。
- `matrix_mul_transpose()`：计算 `A * B^T` 而不显式构造 B 的转置。关键在 `B[j * n + k]`——直接以转置后的行优先顺序访问 B 的元素。这避免了额外的转置操作和内存开销，在 EKF 的 `F * P * F^T` 计算中大量使用。

```c
// 3x3 矩阵求逆（伴随矩阵法）
uint8_t matrix_inv_3x3(const float A[3][3], float inv[3][3])
{
    float det = A[0][0] * (A[1][1] * A[2][2] - A[1][2] * A[2][1])
              - A[0][1] * (A[1][0] * A[2][2] - A[1][2] * A[2][0])
              + A[0][2] * (A[1][0] * A[2][1] - A[1][1] * A[2][0]);

    if (fabsf(det) < 1e-10f) {
        return 0;  // 奇异矩阵
    }
    float inv_det = 1.0f / det;

    inv[0][0] = (A[1][1] * A[2][2] - A[1][2] * A[2][1]) * inv_det;
    inv[0][1] = (A[0][2] * A[2][1] - A[0][1] * A[2][2]) * inv_det;
    inv[0][2] = (A[0][1] * A[1][2] - A[0][2] * A[1][1]) * inv_det;
    inv[1][0] = (A[1][2] * A[2][0] - A[1][0] * A[2][2]) * inv_det;
    inv[1][1] = (A[0][0] * A[2][2] - A[0][2] * A[2][0]) * inv_det;
    inv[1][2] = (A[0][2] * A[1][0] - A[0][0] * A[1][2]) * inv_det;
    inv[2][0] = (A[1][0] * A[2][1] - A[1][1] * A[2][0]) * inv_det;
    inv[2][1] = (A[0][1] * A[2][0] - A[0][0] * A[2][1]) * inv_det;
    inv[2][2] = (A[0][0] * A[1][1] - A[0][1] * A[1][0]) * inv_det;
    return 1;
}
```

**逐行解析：**

- **行列式计算**：按照第一行展开的三阶行列式公式。这是数学上标准的 3×3 行列式展开。
- **奇异性检查**：`fabsf(det) < 1e-10f` 检查行列式是否接近零。如果接近零，矩阵近似奇异，求逆会导致数值爆炸，此时返回 0 表示失败。
- `inv_det = 1.0f / det`：只需要一次除法。
- **伴随矩阵法**：`inv[i][j] = cofactor[j][i] * inv_det`。每个逆矩阵元素是对应余子式（转置后）乘以行列式倒数。
- 注意 `inv[0][1]` 的公式：它使用的是余子式 `C[1][0]`（行列互换），这是伴随矩阵转置的特性。

> **为何不用通用求逆？** 对于 3×3 矩阵，伴随矩阵法只需约 30 次乘法和 1 次除法，远快于 Gauss-Jordan 消元法。EKF 中 S 矩阵始终是 3×3（对应三轴加速度计），因此这个专用函数是最优选择。

---

### 4.7 STM32 主程序与 UART 通信（`main.c`）

```c
int main(void)
{
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART2_UART_Init();

    // 初始化 EKF
    ekf_init();
    // 启动 UART 接收中断（12 字节 = 6 × int16）
    HAL_UART_Receive_IT(&huart2, synthetic_data, DATA_BYTES);

    while (1)
    {
        // 主循环空转，所有处理在中断回调中完成
    }
}
```

**解析 — 主程序结构：**
- `HAL_Init()`：初始化 HAL 库，配置 SysTick（1ms 中断）。
- `SystemClock_Config()`：配置 HSI 8MHz → PLL ×16 → SYSCLK 64MHz。Cortex-M4F 在 64MHz 下有足够的算力执行 EKF。
- `MX_USART2_UART_Init()`：USART2 配置为 115200 baud、8N1、收发模式。
- `ekf_init()`：初始化卡尔曼滤波器——设置 Q/P/R 矩阵并清零状态。
- `HAL_UART_Receive_IT()`：启动 UART 中断接收，等待 12 字节数据。此后主循环空转，所有数据处理在接收完成回调中完成。

```c
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART2)
    {
        // 1. 解包数据（6 × int16, little-endian）
        int16_t gx_raw, gy_raw, gz_raw;
        int16_t ax_raw, ay_raw, az_raw;

        gx_raw = (int16_t)(synthetic_data[0] | (synthetic_data[1] << 8));
        gy_raw = (int16_t)(synthetic_data[2] | (synthetic_data[3] << 8));
        gz_raw = (int16_t)(synthetic_data[4] | (synthetic_data[5] << 8));
        ax_raw = (int16_t)(synthetic_data[6] | (synthetic_data[7] << 8));
        ay_raw = (int16_t)(synthetic_data[8] | (synthetic_data[9] << 8));
        az_raw = (int16_t)(synthetic_data[10] | (synthetic_data[11] << 8));

        // 2. 转换为浮点数（除以比例因子）
        float gx = gx_raw / IMU_SCALE;
        float gy = gy_raw / IMU_SCALE;
        float gz = gz_raw / IMU_SCALE;
        float ax = ax_raw / IMU_SCALE;
        float ay = ay_raw / IMU_SCALE;
        float az = az_raw / IMU_SCALE;

        // 3. EKF 处理
        float dt = 1.0f / IMU_HZ;
        ekf_predict(dt, gx, gy, gz);
        ekf_update_accel(ax, ay, az);

        // 4. 获取滤波结果（弧度）
        float angles[3];
        ekf_get_radians(&angles[0], &angles[1], &angles[2]);

        // 5. 发送结果（3 × float32 = 12 字节）
        HAL_UART_Transmit(&huart2, (uint8_t*)angles, DATA_BYTES, 100);

        // 6. 重新启动接收
        HAL_UART_Receive_IT(&huart2, synthetic_data, DATA_BYTES);
    }
}
```

**逐段深度解析：**

**数据解包（步骤 1）：**
- UART 接收的 12 字节数据包含 6 个 int16 值，采用小端序（little-endian）。
- `synthetic_data[0] | (synthetic_data[1] << 8)`：低字节在低地址（小端序），通过位操作组合为 16 位整数。
- `(int16_t)` 强制转换：确保高位为符号位时正确解释为负数。

**数据转换（步骤 2）：**
- `IMU_SCALE` 是比例因子（Python 端使用 1000.0），将 int16 原始值转换为浮点物理量。
- 陀螺仪数据单位为 rad/s，加速度计数据单位为 m/s²。

**EKF 处理（步骤 3）：**
- `dt = 1.0f / IMU_HZ`：根据采样率计算时间步长。100 Hz → dt = 0.01 秒。
- `ekf_predict(dt, gx, gy, gz)`：先用陀螺仪数据预测新姿态。
- `ekf_update_accel(ax, ay, az)`：再用加速度计数据校正。
- 每次采样执行一次完整的 predict + update 循环。

**结果发送（步骤 4-5）：**
- `ekf_get_radians()` 获取弧度制姿态角。
- `HAL_UART_Transmit()` 直接以 `float*` 强转为 `uint8_t*` 发送 12 字节——利用了 float 和 4 字节在内存中的直接映射关系。这种方法高效但依赖平台浮点字节序。

**中断链重新启动（步骤 6）：**
- `HAL_UART_Receive_IT()` 在回调末尾重新启动接收，形成中断驱动的连续处理循环。
- **关键设计**：所有 EKF 计算在中断上下文中完成。由于 Cortex-M4F 有硬件浮点单元（FPU），6×6 矩阵运算在中断中执行约 10-20 微秒，不会影响系统实时性。

---

### 4.8 Python 仿真与可视化（`imu_uart_simulator.py`）

```python
def generate_data():
    np.random.seed(42)
    t = np.arange(0, DURATION, DT)
    N = len(t)

    # 真实角度（ground truth）
    roll_true = 0.3 * np.sin(0.8 * t) + 0.1 * np.sin(1.2 * t + 0.5)
    pitch_true = 0.2 * np.cos(0.6 * t) + 0.15 * np.sin(0.9 * t + 0.3)
    yaw_true = 0.1 * t + 0.2 * np.sin(0.3 * t)
    true_angles = np.column_stack([roll_true, pitch_true, yaw_true])

    # 陀螺仪 = 真实角速度 + 零偏 + 噪声
    true_gyro = np.gradient(true_angles, DT, axis=0)
    gyro_meas = true_gyro + GYRO_BIAS + np.random.normal(0, GYRO_NOISE_STD, (N, 3))

    # 加速度计 = 重力分量 + 噪声
    ax_true = -G * np.sin(pitch_true)
    ay_true = G * np.cos(pitch_true) * np.sin(roll_true)
    az_true = G * np.cos(pitch_true) * np.cos(roll_true)
    accel_meas = np.column_stack([ax_true, ay_true, az_true]) + \
                 np.random.normal(0, ACCEL_NOISE_STD, (N, 3))

    return t, gyro_meas, accel_meas, true_angles
```

**逐段解析：**

- **真值生成**：使用多频率正弦叠加生成真实姿态角——模拟设备在多轴上的复杂运动。
- **陀螺仪模拟**：`np.gradient` 对真实角度求数值导数得到真实角速度，再叠加恒定零偏 `GYRO_BIAS = [0.015, -0.008, 0.025]` 和高斯白噪声。
- **加速度计模拟**：使用与固件中 `h(x)` 完全相同的公式从真实角度计算重力分量，再叠加噪声。
- **零偏设计**：三轴零偏不同（0.015, -0.008, 0.025），验证 EKF 能独立估计各轴零偏。

```python
def pack(gyro, accel):
    """6 x int16 → 12 字节"""
    gx = int(np.round(gyro[0] * SCALE))
    gy = int(np.round(gyro[1] * SCALE))
    gz = int(np.round(gyro[2] * SCALE))
    ax = int(np.round(accel[0] * SCALE))
    ay = int(np.round(accel[1] * SCALE))
    az = int(np.round(accel[2] * SCALE))
    return struct.pack('<6h', gx, gy, gz, ax, ay, az)

def unpack(data):
    """12 字节 → 3 x float (roll, pitch, yaw)"""
    if len(data) >= 12:
        return struct.unpack('<3f', data[:12])
    return None
```

**解析 — 通信协议：**
- `pack()`：浮点数据乘以 `SCALE=1000` 后取整为 int16，使用 `struct.pack('<6h', ...)` 打包为 12 字节小端序。`<` 表示小端，`6h` 表示 6 个 short（int16）。
- `unpack()`：STM32 返回 3 个 float32（roll, pitch, yaw），使用 `struct.unpack('<3f', ...)` 解包。`3f` 表示 3 个 float。
- **协议对称性**：发送和接收都是 12 字节，但发送是 int16 格式（精度较低），接收是 float32 格式（精度高）。这种非对称设计节省了上行带宽。

```python
# 主循环：发送 → 接收 → 记录
for i in range(N):
    ser.write(pack(gyro[i], accel[i]))     # 发送 IMU 数据
    data = ser.read(12)                     # 接收滤波结果
    if len(data) == 12:
        roll, pitch, yaw = unpack(data)
        roll_filt[i] = roll
        pitch_filt[i] = pitch
        yaw_filt[i] = yaw
    # 计算原始加速度计角度（对比用）
    ax, ay, az = accel[i]
    roll_raw[i] = np.arctan2(ay, az)
    pitch_raw[i] = np.arctan2(-ax, np.sqrt(ay**2 + az**2))
```

**解析 — 实时通信与对比数据：**
- 每次循环发送一个 IMU 样本，等待 STM32 返回滤波结果。
- `roll_raw = arctan2(ay, az)` 和 `pitch_raw = arctan2(-ax, sqrt(ay²+az²))`：从加速度计直接计算角度（无滤波），作为对比基线。这种方法噪声大但无漂移。
- 最终可视化将三条曲线对比：**绿色（真值）** vs **红色虚线（原始加速度计）** vs **蓝色（EKF 滤波）**。

```python
# 误差统计
for i, name in enumerate(['Roll', 'Pitch']):
    err = (true_angles[:, i] - [roll_filt, pitch_filt][i]) * 180/np.pi
    print(f"{name}:")
    print(f"  Mean error: {np.mean(err):.2f}°")
    print(f"  Std dev:    {np.std(err):.2f}°")
    print(f"  Max error:  {np.max(np.abs(err)):.2f}°")
```

**解析 — 结果评估：**
- 误差统计验证 EKF 的准确性：平均误差应接近零（无偏），标准差反映滤波平滑度。
- 从项目提供的滤波结果图可以看到：EKF 输出（蓝色）紧贴真值（绿色），而原始加速度计角度（红色虚线）噪声明显更大。

---

## 五、API 使用指南

### Python 调用示例

```python
import serial
import struct
import numpy as np

# 1. 打开串口
ser = serial.Serial('COM5', 115200, timeout=0.1)
time.sleep(2)  # 等待 STM32 就绪

# 2. 构造 IMU 数据（陀螺仪 rad/s，加速度计 m/s²）
gx, gy, gz = 0.01, -0.03, 0.02    # 角速度
ax, ay, az = 0.12, -0.35, 9.80   # 加速度

# 3. 打包为 12 字节（6 × int16, little-endian）
SCALE = 1000.0
packet = struct.pack('<6h',
    int(round(gx * SCALE)), int(round(gy * SCALE)), int(round(gz * SCALE)),
    int(round(ax * SCALE)), int(round(ay * SCALE)), int(round(az * SCALE)))

# 4. 发送并接收
ser.write(packet)
response = ser.read(12)

# 5. 解包结果（3 × float32）
if len(response) == 12:
    roll, pitch, yaw = struct.unpack('<3f', response)
    print(f"Roll: {np.degrees(roll):.2f}°, Pitch: {np.degrees(pitch):.2f}°, Yaw: {np.degrees(yaw):.2f}°")

ser.close()
```

### cURL 测试（使用虚拟串口）

```bash
# 使用 socat 创建虚拟串口对
socat -d -d pty,raw,echo=0 pty,raw,echo=0

# 发送 12 字节数据到虚拟串口
echo -ne '\x01\x00\x02\x00\x03\x00\x04\x00\x05\x00\x06\x00' > /dev/pts/0

# 接收返回的 12 字节（3 × float32）
cat /dev/pts/1 | xxd | head -1
```

### 嵌入式集成（移植到其他 MCU）

```c
// === 移植到 ESP32 / Arduino 示例 ===
#include "orientation_ekf_rp.h"

void setup() {
    Serial.begin(115200);
    ekf_init();  // 初始化 EKF
}

void loop() {
    // 从 IMU 传感器读取数据
    float gx = readGyroX();  // rad/s
    float gy = readGyroY();
    float gz = readGyroZ();
    float ax = readAccelX();  // m/s²
    float ay = readAccelY();
    float az = readAccelZ();

    // EKF 滤波
    float dt = 0.01f;  // 100 Hz
    ekf_predict(dt, gx, gy, gz);
    ekf_update_accel(ax, ay, az);

    // 获取结果
    float roll, pitch, yaw;
    ekf_get_angles(&roll, &pitch, &yaw);

    // 输出
    Serial.printf("R:%.2f P:%.2f Y:%.2f\n", roll, pitch, yaw);

    delay(10);  // 100 Hz
}
```

> **移植要点**：只需将 `Core/Inc/` 和 `Core/Src/orientation_ekf_rp.c` + `matrix_utils.c` 复制到目标项目，包含头文件即可。核心算法零平台依赖，仅需 C99 标准库的 `math.h`（sinf/cosf/fabsf）和 `string.h`（memset）。

---

## 六、编译与部署

### 环境搭建

| 工具 | 版本 | 用途 |
|------|------|------|
| STM32CubeIDE | 1.14+ | IDE + 编译工具链 |
| ARM GCC | 10.3+ | 命令行编译（可选） |
| STM32CubeMX | 6.10+ | 外设配置生成（可选） |
| Python | 3.8+ | 仿真与可视化 |
| pyserial | 3.5+ | Python 串口通信 |
| numpy | 1.20+ | 数据生成 |
| matplotlib | 3.5+ | 结果绘图 |

### 编译配置

```bash
# 方式一：STM32CubeIDE
# 1. File → Open Projects from File System → 选择项目根目录
# 2. Project → Build All
# 3. 输出：Debug/extended-kalman-filter-stm32.elf + .bin

# 方式二：命令行（ARM GCC）
arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard \
    -DUSE_HAL_DRIVER -DSTM32F303xE \
    -I Core/Inc -I Drivers/STM32F3xx_HAL_Driver/Inc \
    Core/Src/main.c \
    Core/Src/orientation_ekf_rp.c \
    Core/Src/matrix_utils.c \
    Core/Src/stm32f3xx_it.c \
    Core/Src/stm32f3xx_hal_msp.c \
    Core/Src/system_stm32f3xx.c \
    Drivers/STM32F3xx_HAL_Driver/Src/*.c \
    -T STM32F303RETx_FLASH.ld \
    -o ekf_imu.elf -lm

arm-none-eabi-objcopy -O binary ekf_imu.elf ekf_imu.bin
```

### 烧录步骤

```bash
# 方式一：ST-Link（Nucleo 板载）
st-flash write ekf_imu.bin 0x08000000

# 方式二：OpenOCD
openocd -f interface/stlink.cfg -f target/stm32f3x.cfg \
    -c "program ekf_imu.elf verify reset exit"

# 方式三：STM32CubeProgrammer
STM32_Programmer_CLI -c port=swd -w ekf_imu.bin 0x08000000 -v
```

### 运行验证

```bash
# 1. 安装 Python 依赖
pip install pyserial numpy matplotlib

# 2. 查看串口设备
# Linux: ls /dev/ttyUSB*
# Windows: 设备管理器 → 端口 (COM 和 LPT)

# 3. 修改 imu_uart_simulator.py 中的端口
PORT = '/dev/ttyUSB0'  # 或 'COM5'

# 4. 运行仿真
python imu_uart_simulator.py

# 预期输出：
# Generisanje podataka...
# 1000 uzoraka
# Slanje podataka...
# 100/1000
# 200/1000
# ...
# STATISTIKA
# Roll:
#   Mean error: 0.12°
#   Std dev:    0.34°
#   Max error:  1.23°
```

---

## 七、项目亮点与适用场景

### 项目亮点

| 亮点 | 说明 |
|------|------|
| **数学完备性** | 完整实现 EKF 五大方程：状态预测、协方差预测、卡尔曼增益、状态更新、协方差更新（Joseph 形式） |
| **零偏在线估计** | 将陀螺仪三轴零偏作为状态变量，通过加速度计测量间接校正，无需静态校准 |
| **雅可比手动推导** | 3×6 测量雅可比矩阵完全手动推导并逐元素注释，数学推导过程清晰可验证 |
| **平台无关核心** | EKF + 矩阵库仅依赖 C99 标准库，可一行不改移植到任意 MCU |
| **完整验证闭环** | Python 生成带已知零偏和噪声的合成数据 → UART 实时传输 → STM32 EKF 处理 → 结果回传 → 可视化对比 |
| **Joseph 形式协方差** | 使用数值稳定的协方差更新公式，保证长时间运行不发散 |
| **中断驱动架构** | UART 中断回调中完成完整 EKF 计算，主循环空转，极低延迟 |
| **3×3 伴随矩阵求逆** | 针对固定维度矩阵的解析求逆，比通用数值方法快一个数量级 |

### 适用场景

| 场景 | 描述 |
|------|------|
| **无人机/平衡车姿态控制** | Roll/Pitch 估计是飞控和平衡控制的核心输入 |
| **机器人关节角度监测** | IMU 安装在机械臂关节处，实时监测关节角度 |
| **运动追踪设备** | 可穿戴设备中的姿态追踪，如健身动作识别 |
| **相机防抖** | EKF 滤波后的低噪声角度信号可直接驱动云台稳定 |
| **教学与学习** | 逐行注释的 EKF 实现是学习卡尔曼滤波的最佳实践教材 |
| **传感器融合研究** | 作为基础框架，扩展磁力计实现 9-DOF、或添加 GPS 实现组合导航 |

---

## 八、总结

DjVul 的 Extended-Kalman-Filter---STM32 项目虽仅有数天历史，却以极其清晰的代码结构和详尽注释，完整呈现了从数学推导到嵌入式实现的 EKF 全流程。项目最大的价值在于 **不回避数学**——雅可比矩阵的每个元素都有推导依据，Joseph 形式协方差更新体现了数值稳定性的工程考量，而 Python 仿真闭环则让验证变得可视化、可量化。

从架构角度看，作者将平台无关的算法核心（EKF + 矩阵库）与平台相关的 HAL 层（UART、GPIO）干净分离，使核心代码可以零修改移植到任意 MCU。中断驱动的处理架构在 64MHz Cortex-M4F 上绰绰有余，为集成更多功能（如磁力计融合、卡尔曼滤波器多级架构）留出了充足的算力空间。

对于嵌入式开发者而言，这个项目是一份难得的 **从理论到工程** 的完整参考——不仅展示了"怎么做"，更解释了"为什么这么做"。

---

📝 作者：蔡浩宇（jun-chy）
📅 日期：2026-07-11
🔗 项目地址：[https://github.com/DjVul/Extended-Kalman-Filter---STM32](https://github.com/DjVul/Extended-Kalman-Filter---STM32)
