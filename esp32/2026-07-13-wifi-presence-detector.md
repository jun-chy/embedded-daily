# ESP32-C3 Wi-Fi CSI 混合存在雷达深度解析 — 无摄像头/PIR 的人体存在检测方案

> **日期：** 2026-07-13  
> **分类：** ESP32 / 嵌入式 IoT  
> **Stars：** 1  
> **作者：** 蔡浩宇（jun-chy）

---

## 项目链接

| 属性 | 信息 |
|------|------|
| GitHub 地址 | [KavyaSinghal05/wifi-presence-detector](https://github.com/KavyaSinghal05/wifi-presence-detector) |
| 作者 | KavyaSinghal05 |
| 许可证 | MIT License |
| 最近更新 | 2026-07-13 |
| 主要语言 | C |
| 目标平台 | ESP32-C3 |
| 开发框架 | ESP-IDF v5.4+ |

---

## 项目简介

这是一个运行在 ESP32-C3 上的**生产级人体存在检测固件**，利用 Wi-Fi 信道状态信息（CSI）实现房间内的人体存在检测——无需摄像头、无需 PIR 传感器。项目的设计初衷是将固件嵌入空调内部，当房间无人时自动关闭空调，有人时保持运行（无论此人在走动、静坐还是仅仅是呼吸）。

该固件融合了两条独立的信号链——**幅度偏差路径**和**微多普勒脉冲对路径**，并采用了 12 项高级精度增强技术，达到了生产级占用检测水平。

### 核心特性一览

| # | 特性 | 说明 |
|:-:|------|------|
| 1 | 混合引擎融合 | 幅度 + 多普勒双链路互补，覆盖各自盲区 |
| 2 | Hampel 子载波滤波 | 抑制硬件噪声，保留真实运动信号 |
| 3 | Hann 加窗重叠 FFT | 降低 6dB 频谱泄漏，2 倍多普勒更新速率 |
| 4 | CFAR 多普勒检测 | 每个频率bin自适应阈值 |
| 5 | 卡尔曼滤波存在度量 | 自适应平滑：快速进入、稳定保持 |
| 6 | 共模漂移补偿 | 消除环境基线偏移 |
| 7 | IIR 带通呼吸检测 | 0.15-0.5Hz 隔离，减少 HVAC 误报 |
| 8 | 相位相干性指数 | 通过空间相关性检测静止人员 |
| 9 | 在线重新校准 | 空闲状态下阈值自适应更新 |
| 10 | 对齐式人数统计 | 无量纲、尺度无关、自学习 |
| 11 | NVS 双路径校准 | 疑似检测防止污染校准 |
| 12 | HT-LTF + STBC 启用 | 802.11n 帧上 +3dB SNR |

---

## 硬件架构

### 系统组成图

```
┌─────────────────────────────────────────────────────────┐
│                    ESP32-C3 开发板                       │
│                                                         │
│  ┌───────────┐    ┌──────────────┐    ┌─────────────┐  │
│  │  Wi-Fi    │───▶│  CSI 引擎    │───▶│  信号处理   │  │
│  │  天线     │    │  (LLTF+      │    │  双链路融合 │  │
│  │  (内置)   │    │   HT-LTF+    │    │  (幅度+     │  │
│  │           │    │   STBC)      │    │   多普勒)   │  │
│  └───────────┘    └──────────────┘    └──────┬──────┘  │
│                                               │         │
│                          ┌────────────────────┘         │
│                          ▼                              │
│                   ┌─────────────┐     ┌──────────┐      │
│                   │  FSM 状态机  │────▶│  GPIO 4  │──────┼──▶ AC 控制板
│                   │  (4 态)     │     │  存在输出 │      │
│                   └─────────────┘     └──────────┘      │
│                          │                              │
│                   ┌──────┴──────┐                       │
│                   │  NVS Flash  │                       │
│                   │  (校准持久化)│                       │
│                   └─────────────┘                       │
│                                                         │
│  ┌──────────────────────────────────┐                   │
│  │  UART 串口 (日志 + PLOT 遥测)     │                   │
│  └──────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────┘

         ┌─────────────┐
         │  Wi-Fi 路由器 │◀─── Ping + DNS 流量生成
         │  (AP)        │     (主动激发 CSI 数据包)
         └─────────────┘
```

### 引脚配置

| GPIO | 功能 | 说明 |
|------|------|------|
| GPIO 4 | 存在输出 | HIGH=有人(校准期间默认), LOW=确认无人 |
| -1 | 禁用输出 | 通过 menuconfig 设置 |

### 硬件清单

| 组件 | 型号 | 数量 | 说明 |
|------|------|:----:|------|
| 主控 | ESP32-C3 开发板 | 1 | 任何 ESP32-C3 变体 |
| Wi-Fi 路由器 | 任意 2.4GHz | 1 | 作为 AP 激发 CSI |
| AC 控制板 | — | 1 | 接收 GPIO 高低电平 |
| 杜邦线 | — | 2 | 连接 GPIO 4 到 AC 控制板 |

---

## 固件架构

### 文件结构

| 文件路径 | 大小 | 职责 |
|----------|------|------|
| `CMakeLists.txt` | 377B | 项目顶层构建配置，启用最小构建 |
| `main/CMakeLists.txt` | 382B | main 组件构建配置 |
| `main/Kconfig.projbuild` | 780B | Wi-Fi 凭据 + 占用 GPIO 配置菜单 |
| `main/ac_presence_main.c` | 83KB | **核心固件**：混合雷达引擎、幅度+多普勒融合、FSM |
| `main/fft.c` | 1.8KB | Radix-2 Cooley-Tukey FFT 实现 |
| `main/fft.h` | 141B | FFT 接口声明 |
| `main/hello_world_main.c` | 37KB | 遗留多普勒-only 引擎（参考用） |
| `tools/plot_presence.py` | 5.5KB | 桌面端实时串口绘图工具 |
| `pytest_hello_world.py` | 1.9KB | pytest 测试脚本 |

### 模块职责

| 模块 | 职责 |
|------|------|
| CSI 接收回调 | 从 Wi-Fi 驱动快速拷贝 CSI 数据，MAC 过滤、限速，推入环形缓冲 |
| 信号处理任务 | 从环形缓冲取包，执行 13 步流水线处理 |
| 幅度路径 | AGC 归一化 → 基线偏差 → Hampel 滤波 → 漂移补偿 → 卡尔曼平滑 |
| 多普勒路径 | 相位解缠 → 线性去趋势 → 脉冲对自相关 → 跨子载波聚合 |
| FFT + CFAR | Hann 加窗重叠 FFT → 幅度谱 → CFAR 自适应峰值检测 |
| 呼吸检测 | 双 EWMA 带通 (0.15-0.5Hz) → 方差窗口判定 |
| FSM 状态机 | 4 态有限状态机：校准→空闲↔占用移动↔占用静止 |
| NVS 校准 | 稳健统计 (中位数+MAD) 校准，Flash 持久化，疑似检测 |
| 流量生成器 | Ping + DNS 双路径激发 CSI 数据包 |
| Python 绘图器 | 串口读取 PLOT 遥测，matplotlib 实时滚动图表 |

---

## 核心代码深度分析

### 1. 应用入口 — `app_main()`

```c
void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());
    wifi_event_group = xEventGroupCreate();
    csi_ring_buf = xRingbufferCreate(CSI_BUFFER_SIZE, RINGBUF_TYPE_NOSPLIT);

    presence_output_init();

    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
```

**逐行解析：**
- `nvs_flash_init()`：初始化非易失性存储，用于持久化校准阈值。即使断电重启，上次校准的良好阈值也能从 Flash 中恢复。
- `xEventGroupCreate()`：创建事件组，用于 Wi-Fi 连接状态的同步。当获得 IP 地址时设置 `WIFI_CONNECTED_BIT`，其他任务通过等待该 bit 来决定是否开始工作。
- `xRingbufferCreate(CSI_BUFFER_SIZE, RINGBUF_TYPE_NOSPLIT)`：创建 32KB 环形缓冲区。`NOSPLIT` 类型确保每个 CSI 数据包作为一个完整的不可分割项存储，不会被拆分到缓冲区的两端。
- `presence_output_init()`：初始化占用输出 GPIO，默认拉高（安全默认=有人）。
- `esp_netif_init()` + `esp_event_loop_create_default()`：初始化网络接口和默认事件循环，是 Wi-Fi 连接的基础设施。

```c
    wifi_config_t wifi_config = {
        .sta = {
            .ssid = CONFIG_WIFI_SSID,
            .password = CONFIG_WIFI_PASSWORD,
            .scan_method = WIFI_ALL_CHANNEL_SCAN,
            .sort_method = WIFI_CONNECT_AP_BY_SIGNAL,
            .threshold.rssi = -127,
            .pmf_cfg = { .capable = true, .required = false },
        },
    };

    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &wifi_config));
    ESP_ERROR_CHECK(esp_wifi_start());
```

**逐行解析：**
- `WIFI_ALL_CHANNEL_SCAN`：全信道扫描而非快速扫描，确保找到信号最强的 AP。这对 CSI 质量至关重要——弱的信号会导致 I/Q 数据噪声增大。
- `threshold.rssi = -127`：设置极低的 RSSI 阈值（-127dBm 几乎是最低值），意味着不拒绝任何 AP。这是有意为之：项目需要连接到特定路由器获取 CSI，信号强度不是连接的硬性门槛。
- `pmf_cfg.capable = true`：支持 PMF（Protected Management Frames），但不强制要求。兼容现代路由器的安全特性。

```c
    // CSI config: enable LLTF + HT-LTF + STBC for maximum SNR
    wifi_csi_config_t csi_cfg;
    memset(&csi_cfg, 0, sizeof(wifi_csi_config_t));
    csi_cfg.lltf_en           = true;
    csi_cfg.htltf_en          = true;     // +3 dB SNR on HT packets
    csi_cfg.stbc_htltf2_en    = true;     // STBC second HT-LTF if available
    csi_cfg.channel_filter_en = true;

    ESP_ERROR_CHECK(esp_wifi_set_csi_config(&csi_cfg));
    ESP_ERROR_CHECK(esp_wifi_set_csi_rx_cb(csi_rx_callback, NULL));

    esp_log_level_set("ping_sock", ESP_LOG_NONE);

    xTaskCreate(csi_processing_task, "hybrid_radar", 32768, NULL, 5, NULL);
    xTaskCreate(dns_traffic_task, "dns_traffic", 3072, NULL, 3, NULL);
}
```

**逐行解析：**
- `lltf_en = true`：启用 Legacy LTF（Long Training Field）CSI 提取，这是 802.11a/g 的标准训练字段，提供基本的子载波 CSI。
- `htltf_en = true`：启用 HT-LTF（High Throughput LTF），这是 802.11n 引入的额外训练字段，可以在 HT 帧上提供 +3dB SNR 的 CSI 数据——显著提升信号质量。
- `stbc_htltf2_en = true`：启用 STBC（Space-Time Block Coding）的第二个 HT-LTF。当 AP 使用 STBC 时，可利用额外的训练字段进一步提升 CSI 可靠性。
- `channel_filter_en = true`：只接收当前关联信道的 CSI，过滤其他信道的噪声包。
- `xTaskCreate(csi_processing_task, ..., 32768, ...)`：创建信号处理任务，分配 32KB 栈空间。这是因为处理任务中使用了大量静态数组（FFT 缓冲、校准样本等），需要足够的栈空间。
- `xTaskCreate(dns_traffic_task, ..., 3072, ...)`：创建 DNS 流量生成任务，仅 3KB 栈——该任务只是循环发送 DNS 查询，逻辑简单。

---

### 2. CSI 接收回调 — 快速拷贝，不做处理

```c
static void csi_rx_callback(void *ctx, wifi_csi_info_t *info)
{
    static int64_t last_accept_us = 0;

    if (!info || !info->buf || info->len == 0) return;
    csi_rx_total++;

    if (!ap_bssid_valid || memcmp(info->mac, ap_bssid, sizeof(ap_bssid)) != 0) {
        csi_rx_foreign++;
        memcpy(csi_last_foreign_mac, info->mac, sizeof(csi_last_foreign_mac));
        return;
    }

    int64_t now = esp_timer_get_time();
    if (now - last_accept_us < CSI_MIN_SPACING_US) {
        csi_rx_ratelimited++;
        return;
    }
    last_accept_us = now;

    csi_packet_t packet;
    packet.ts_us = now;
    packet.rssi = info->rx_ctrl.rssi;
    packet.len = (info->len > MAX_CSI_LEN) ? MAX_CSI_LEN : info->len;
    memcpy(packet.payload, info->buf, packet.len);

    xRingbufferSend(csi_ring_buf, &packet, sizeof(packet), 0);
}
```

**逐段解析：**

这个回调运行在 Wi-Fi 驱动任务上下文中，设计原则是**尽可能快地拷贝数据然后返回**——任何耗时操作都会导致 CSI 数据包丢失。

- **MAC 地址过滤**（`memcmp(info->mac, ap_bssid, ...)`）：只接收来自关联 AP 的 CSI 数据包。其他设备的 Wi-Fi 帧虽然也携带 CSI，但来自不同位置/天线，会引入噪声。通过 BSSID 过滤确保所有 CSI 来自同一 AP，保证信道一致性。
- **速率限制**（`now - last_accept_us < CSI_MIN_SPACING_US`）：限制接受频率为约 22Hz（45ms 间隔）。这个速率是精心选择的——足够多普勒路径形成有效的脉冲对（需要 ≥10Hz），又不会过载处理任务。如果 ping 间隔为 25ms（40Hz），限速器会将实际处理速率降至 ~20Hz。
- **时间戳记录**（`packet.ts_us = now`）：使用微秒级时间戳。多普勒路径需要精确的 Δt 来计算多普勒频移，时间戳的精度直接影响频率估计的准确性。
- **零等待发送**（`xRingbufferSend(..., 0)`）：超时参数为 0，即如果缓冲区满就丢弃。宁可丢包也不能阻塞 Wi-Fi 驱动任务。

---

### 3. I/Q 提取与 AGC 归一化

```c
// STEP 1 — Extract I/Q, compute magnitude and phase per subcarrier
// ESP-IDF CSI byte order: [imaginary (Q), real (I)] per subcarrier.
float mag_slot[MAX_TRACKED_SC];
float phi_slot[MAX_TRACKED_SC];
float raw_mag_full[64];
bool  ok_slot[MAX_TRACKED_SC];
int   ok_count = 0;
float mag_sum = 0.0f;

for (int s = 0; s < sc_count; s++) {
    int k = sc_index[s];
    ok_slot[s] = false;
    mag_slot[s] = 0.0f;
    phi_slot[s] = 0.0f;
    if (k >= pairs_in_packet) continue;

    float Q = (float)raw[2 * k];
    float I = (float)raw[2 * k + 1];
    float m = sqrtf(I * I + Q * Q);
    if (m < 1.0f) continue;         // Nulled/dead subcarrier

    mag_slot[s] = m;
    phi_slot[s] = atan2f(Q, I);
    raw_mag_full[k] = m;
    ok_slot[s] = true;
    ok_count++;
    mag_sum += m;
}

if (ok_count < MIN_VALID_SC) {
    memset(pp_prev_ok, 0, sizeof(pp_prev_ok));
    vRingbufferReturnItem(csi_ring_buf, packet);
    continue;
}

float inv_mean_mag = (float)ok_count / mag_sum;   // AGC normalisation
```

**逐段解析：**

- **子载波计划**：ESP32 的 20MHz 信道有 64 个子载波，但项目只使用索引 6-31 和 33-58（共 52 个子载波），跳过了 DC 子载波（索引 32）和边缘保护子载波。这些被跳过的子载波不携带有效信息。
- **I/Q 提取**：ESP-IDF 的 CSI 数据格式是 `[Q, I]` 交替排列。`raw[2*k]` 是虚部 Q，`raw[2*k+1]` 是实部 I。注意字节顺序——搞反 I 和 Q 会导致相位计算完全错误。
- **幅度计算**（`sqrtf(I*I + Q*Q)`）：向量的模长，表示该子载波的信号强度。如果 `m < 1.0f`，说明该子载波被 nulling（置零）或不可用，直接跳过。
- **相位计算**（`atan2f(Q, I)`）：四象限反正切，返回 [-π, π] 范围的相位角。相位信息是多普勒路径的核心——人体运动导致的相位变化是检测的依据。
- **有效子载波检查**（`ok_count < MIN_VALID_SC`）：如果有效子载波少于 16 个（共 52 个），说明这包 CSI 质量太差，直接丢弃。同时清除多普勒路径的 `pp_prev_ok`，防止使用过期的前一包数据。
- **AGC 归一化**（`inv_mean_mag = ok_count / mag_sum`）：计算所有有效子载波平均幅度的倒数。后续用 `norm_mag = raw_mag_full[k] * inv_mean_mag` 将每个子载波的幅度归一化到约 1.0。这消除了 RSSI 波动导致的整体幅度变化，使得偏差检测只关注子载波间的相对变化。

---

### 4. 相位解缠与线性去趋势

```c
// STEP 2 — Phase unwrap + linear detrend (remove CFO + timing offset)
{
    float prev_u = 0.0f;
    bool first = true;
    float n = 0.0f, Sx = 0.0f, Sy = 0.0f, Sxy = 0.0f, Sxx = 0.0f;

    for (int s = 0; s < sc_count; s++) {
        if (!ok_slot[s]) continue;
        if (first) { first = false; }
        else { phi_slot[s] = prev_u + wrap_pm_pi(phi_slot[s] - prev_u); }
        prev_u = phi_slot[s];

        float x = (float)sc_index[s];
        n += 1.0f; Sx += x; Sy += phi_slot[s];
        Sxy += x * phi_slot[s]; Sxx += x * x;
    }

    float denom = n * Sxx - Sx * Sx;
    float b = (fabsf(denom) > 1e-6f) ? (n * Sxy - Sx * Sy) / denom : 0.0f;
    float a = (n > 0.0f) ? (Sy - b * Sx) / n : 0.0f;

    for (int s = 0; s < sc_count; s++) {
        if (!ok_slot[s]) continue;
        phi_slot[s] -= (a + b * (float)sc_index[s]);
    }
}
```

**逐段解析：**

这段代码处理多普勒路径最关键的前置步骤——将原始相位转换为可用于脉冲对相关的稳定信道属性。

- **相位解缠**（`wrap_pm_pi(phi_slot[s] - prev_u)`）：相邻子载波的相位差可能超过 ±π，导致 2π 跳变。`wrap_pm_pi` 函数将差值折叠到 [-π, π] 范围内，然后累加到前一个解缠后的值上。这消除了相位跳变，得到连续的相位序列。

- **最小二乘线性去趋势**：代码在遍历过程中累积统计量 `n, Sx, Sy, Sxy, Sxx`，然后计算线性回归的斜率 `b` 和截距 `a`：
  - 斜率 `b`：反映载波频率偏移（CFO）——收发双方的晶振不完全同步，导致相位随子载波索引线性变化。
  - 截距 `a`：反映定时偏移——采样时刻的偏差导致所有子载波的相位有一个整体偏移。
  - 减去 `a + b * sc_index[s]` 后，剩余的相位只包含人体运动引起的变化——这正是多普勒路径需要的信息。

- **为何重要**：如果不做去趋势，CFO 和定时偏移会在后续的脉冲对自相关中引入虚假的多普勒信号，导致空房间被误判为有人。

---

### 5. 幅度路径 — 基线偏差与 Hampel 滤波

```c
// STEP 3a — AMPLITUDE PATH: normalised deviations from frozen baseline
for (int s = 0; s < sc_count; s++) {
    sub_deviations[s] = 0.0f;
    if (!ok_slot[s]) continue;

    int k = sc_index[s];
    float norm_mag = raw_mag_full[k] * inv_mean_mag;

    if (warming_up) {
        baseline_sum[k] += norm_mag;
        baseline_cnt[k]++;
        prev_phase_saved[s] = phi_slot[s];
        continue;
    }

    if (frozen_baseline[k] <= 0.0f) {
        frozen_baseline[k] = norm_mag;
        continue;
    }

    float deviation = fabsf(norm_mag - frozen_baseline[k]);
    sub_deviations[valid_sc_count] = deviation;
    total_deviation += deviation;
    valid_sc_count++;

    // Phase velocity for direction estimation
    float pdelta = wrap_pm_pi(phi_slot[s] - prev_phase_saved[s]);
    net_phase_velocity += pdelta;
    phase_vel_count++;
    phase_delta_slot[s] = pdelta;
    prev_phase_saved[s] = phi_slot[s];

    // Baseline adaptation (only when confirmed EMPTY)
    if (adapt_baseline) {
        frozen_baseline[k] = alpha_bl * norm_mag + (1.0f - alpha_bl) * frozen_baseline[k];
    }
}
```

**逐段解析：**

- **冻结基线**（`frozen_baseline`）：在预热阶段（前 20 个包）计算每个子载波的平均归一化幅度作为基线。预热完成后基线"冻结"——只在确认 EMPTY 状态下才缓慢更新（`ALPHA_FILTER = 0.02`），防止人的存在污染基线。
- **偏差计算**（`fabsf(norm_mag - frozen_baseline[k])`）：每个子载波当前幅度与基线的绝对偏差。人体存在会改变多径环境，导致某些子载波幅度显著变化。
- **相位速度**（`wrap_pm_pi(phi_slot[s] - prev_phase_saved[s])`）：计算同一子载波在相邻两个 CSI 包之间的相位变化量。后续用于估计运动方向（接近/远离）。
- **基线自适应**（`alpha_bl * norm_mag + (1-alpha_bl) * frozen_baseline[k]`）：指数移动平均更新基线。正常状态下 `alpha = 0.02`（很慢），重新基线化后 `alpha = 0.10`（快 5 倍），快速适应新的空房间环境。

```c
// STEP 3b — Hampel outlier filter on subcarrier deviations
static void hampel_segment(const float *d, int start, int end,
                           float *out_sum, int *out_n)
{
    for (int i = start; i < end; i++) {
        int lo = (i - HAMPEL_HALF_WIN < start) ? start : i - HAMPEL_HALF_WIN;
        int hi = (i + HAMPEL_HALF_WIN >= end)  ? end - 1 : i + HAMPEL_HALF_WIN;

        float local[5];
        int cnt = 0;
        for (int j = lo; j <= hi; j++) local[cnt++] = d[j];

        // Insertion sort (cnt <= 5)
        for (int a = 1; a < cnt; a++) {
            float key = local[a];
            int b = a - 1;
            while (b >= 0 && local[b] > key) { local[b + 1] = local[b]; b--; }
            local[b + 1] = key;
        }

        float med = local[cnt / 2];
        float abs_devs[5];
        for (int j = 0; j < cnt; j++) abs_devs[j] = fabsf(local[j] - med);
        // Sort abs_devs...
        float local_mad = abs_devs[cnt / 2];
        float sigma_local = 1.4826f * local_mad;

        float val = (fabsf(d[i] - med) > HAMPEL_K * sigma_local) ? med : d[i];
        *out_sum += val;
        (*out_n)++;
    }
}
```

**逐段解析：**

Hampel 滤波器是一种基于中位数的鲁棒异常值检测方法，特别适合嵌入式环境：

- **滑动窗口**：对每个子载波，取其前后各 2 个邻居（5 点窗口），计算局部中位数和 MAD（中位绝对偏差）。
- **异常值替换**：如果某子载波的偏差超过 `HAMPEL_K * sigma_local`（3 倍局部标准差），就用局部中位数替换。这有效剔除了硬件噪声尖峰，同时保留了真实运动引起的全局偏差。
- **MAD → σ 转换**（`1.4826f * local_mad`）：对于高斯分布，MAD 与标准差的关系是 σ ≈ 1.4826 × MAD。这个系数使得 MAD 可以作为标准差的鲁棒替代。
- **分半处理**（`hampel_segment` 被调用两次，分别处理上下半频带）：ESP32 的 CSI 子载波分为正负频率两组，它们的噪声特性不同。分别处理避免跨频带的异常值污染。
- **插入排序**（cnt ≤ 5）：窗口仅 5 个元素，插入排序比 qsort 更高效——没有函数调用开销，对缓存友好。

---

### 6. 共模漂移补偿与卡尔曼滤波

```c
// STEP 3c — Common-mode drift compensation
float min_dev = 1e9f;
for (int s = 0; s < valid_sc_count; s++) {
    if (sub_deviations[s] < min_dev) min_dev = sub_deviations[s];
}
raw_presence -= min_dev * 0.8f;
if (raw_presence < 0.0f) raw_presence = 0.0f;

// STEP 3d — Kalman-filtered presence metric
smooth_presence_metric = kalman_update(&kalman_presence, raw_presence);
```

```c
// Scalar Kalman filter
typedef struct {
    float x;     // State estimate
    float P;     // Estimate uncertainty
} kalman1d_t;

static inline float kalman_update(kalman1d_t *kf, float measurement)
{
    float P_pred = kf->P + KALMAN_Q;           // Q = 0.001 (process noise)
    float K = P_pred / (P_pred + KALMAN_R);     // R = 0.01 (measurement noise)
    kf->x = kf->x + K * (measurement - kf->x);
    kf->P = (1.0f - K) * P_pred;
    return kf->x;
}
```

**逐段解析：**

- **共模漂移补偿**：找出所有子载波中的最小偏差 `min_dev`，然后从总偏差中减去 `min_dev * 0.8`。原理是：人体不可能均匀地影响所有子载波（这违反了多径传播的物理特性）。如果所有子载波都有类似大小的偏差，那一定是环境漂移（如温度变化、天线方向变化），而非人体存在。系数 0.8（而非 1.0）保留了少量余量，避免过度削减真实信号。

- **标量卡尔曼滤波器**：
  - `Q = 0.001`（过程噪声）：极小值，表示真实状态变化缓慢——人不会瞬移。
  - `R = 0.01`（测量噪声）：相对较大，表示单次测量有较多噪声。
  - 卡尔曼增益 `K = P_pred / (P_pred + R)`：当预测不确定性 `P_pred` 大时（刚进入新状态），K 接近 1，滤波器快速跟随测量值——实现"快速进入"。当 `P_pred` 小时（状态稳定），K 很小，滤波器平滑测量值——实现"稳定保持"。
  - 这种自适应行为正是存在检测需要的：人进入房间时快速响应，人静止时稳定不抖。

---

### 7. 呼吸检测 — 双 EWMA 带通滤波器

```c
// STEP 4 — Breathing bandpass (dual-EWMA)
breath_fast = BREATH_FAST_ALPHA * smooth_presence_metric
            + (1.0f - BREATH_FAST_ALPHA) * breath_fast;   // alpha = 0.15
breath_slow = BREATH_SLOW_ALPHA * smooth_presence_metric
            + (1.0f - BREATH_SLOW_ALPHA) * breath_slow;   // alpha = 0.03
breathing_signal = breath_fast - breath_slow;

breathing_window[breathing_win_idx] = breathing_signal;
breathing_win_idx = (breathing_win_idx + 1) % BREATHING_WINDOW;
if (breathing_fill < BREATHING_WINDOW) breathing_fill++;

breathing_ready = (breathing_fill >= BREATHING_MIN_FILL);
breathing_variance = 0.0f;
if (breathing_ready) {
    breathing_variance = compute_variance(breathing_window, breathing_fill, NULL);
}
```

**逐段解析：**

这是一个巧妙的**双指数移动平均（EWMA）带通滤波器**，用极低的计算成本实现了 0.15-0.5Hz 的带通效果：

- **快速 EWMA**（`alpha = 0.15`）：跟踪高达约 0.6Hz 的信号变化——覆盖呼吸频率（0.15-0.5Hz，即 9-30 次/分钟）。
- **慢速 EWMA**（`alpha = 0.03`）：只跟踪低于约 0.1Hz 的信号变化——反映基线漂移和缓慢环境变化。
- **差值**（`breath_fast - breath_slow`）：减去慢速趋势后，剩余的就是 0.1-0.6Hz 范围内的信号——正好是呼吸频带。
- **方差窗口**（100 个样本，约 5 秒）：在 5 秒窗口内计算呼吸信号的方差。静坐呼吸的人会产生微小的周期性幅度变化，这些变化在方差上体现为高于空房间的值。
- **最小填充量**（`BREATHING_MIN_FILL = 30`）：至少积累 30 个样本（1.5 秒）才开始计算方差，避免初始瞬态导致误判。

相比传统 IIR/FIR 滤波器，双 EWMA 方案仅需 2 个浮点状态变量，非常适合 RAM 有限的 ESP32-C3。

---

### 8. FFT + CFAR 多普勒峰值检测

```c
// STEP 6 — Overlapping Hann-windowed FFT + CFAR peak detection
doppler_circ[doppler_write] = raw_presence;
doppler_write = (doppler_write + 1) % FFT_SIZE;
doppler_since_fft++;

if (doppler_since_fft >= FFT_OVERLAP) {
    float fft_re[FFT_SIZE], fft_im[FFT_SIZE];

    // Copy circular buffer to linear + Hann window + DC removal
    float dc_sum = 0.0f;
    for (int i = 0; i < FFT_SIZE; i++) {
        int idx = (doppler_write + i) % FFT_SIZE;
        dc_sum += doppler_circ[idx];
    }
    float dc_mean = dc_sum / FFT_SIZE;
    for (int i = 0; i < FFT_SIZE; i++) {
        int idx = (doppler_write + i) % FFT_SIZE;
        float hann = 0.5f * (1.0f - cosf(2.0f * (float)M_PI * i / (FFT_SIZE - 1)));
        fft_re[i] = (doppler_circ[idx] - dc_mean) * hann;
        fft_im[i] = 0.0f;
    }

    fft_compute(fft_re, fft_im, FFT_SIZE);

    // Compute magnitude spectrum for CFAR
    float mag_spectrum[FFT_SIZE / 2];
    for (int i = 1; i < FFT_SIZE / 2; i++) {
        mag_spectrum[i] = sqrtf(fft_re[i] * fft_re[i] + fft_im[i] * fft_im[i]);
    }
    mag_spectrum[0] = 0.0f;

    latest_cfar_peaks = cfar_detect_peaks(mag_spectrum, FFT_SIZE / 2,
                                          DOPPLER_BAND_MIN_BIN,
                                          DOPPLER_BAND_MAX_BIN);
    doppler_since_fft = 0;
}
```

**逐段解析：**

- **循环缓冲区**（`doppler_circ[FFT_SIZE]`）：256 点循环缓冲区存储原始存在度量。每收到一个包写入一个值，当累积了 `FFT_OVERLAP = 128` 个新值（50% 重叠）时执行一次 FFT。
- **DC 去除**（`dc_mean`）：从所有样本中减去直流分量。存在度量的直流分量代表平均幅度水平，不是多普勒信息——去除它避免零频率bin淹没其他bin。
- **Hann 窗**（`0.5 * (1 - cos(2πi/(N-1)))`）：加窗减少频谱泄漏。Hann 窗的旁瓣衰减为 -31.5dB（相比矩形窗的 -13dB），可以更清晰地区分相邻的多普勒频率。代码注释中提到"-6dB 泄漏"是指 Hann 窗的主瓣宽度。
- **FFT 计算**（`fft_compute(fft_re, fft_im, FFT_SIZE)`）：调用自实现的 Radix-2 Cooley-Tukey FFT。256 点 FFT 在 ESP32-C3 上只需几毫秒。
- **幅度谱**（`sqrtf(re*re + im*im)`）：取前 N/2 个 bin（实信号频谱对称），计算每个频率bin的幅度。
- **CFAR 检测**：将幅度谱传递给 CFAR 检测器，在多普勒频带（bin 2-26，约 0.16-2Hz）内寻找峰值。

```c
// CFAR Doppler peak detection
static int cfar_detect_peaks(const float *mag, int n_bins,
                             int band_min, int band_max)
{
    int peaks = 0;
    if (band_max > n_bins - 2) band_max = n_bins - 2;

    for (int i = band_min; i <= band_max; i++) {
        float noise_sum = 0.0f;
        int noise_count = 0;

        for (int j = i - CFAR_GUARD_CELLS - CFAR_TRAIN_CELLS;
             j < i - CFAR_GUARD_CELLS; j++) {
            if (j >= 1 && j < n_bins) { noise_sum += mag[j]; noise_count++; }
        }
        for (int j = i + CFAR_GUARD_CELLS + 1;
             j <= i + CFAR_GUARD_CELLS + CFAR_TRAIN_CELLS; j++) {
            if (j >= 1 && j < n_bins) { noise_sum += mag[j]; noise_count++; }
        }

        if (noise_count > 0) {
            float local_thr = CFAR_ALPHA * (noise_sum / noise_count);
            if (mag[i] > local_thr && mag[i] > mag[i - 1] && mag[i] > mag[i + 1]) {
                peaks++;
            }
        }
    }
    return peaks;
}
```

**逐段解析：**

CFAR（Constant False Alarm Rate，恒虚警率）是雷达领域的经典算法，这里用于多普勒峰值检测：

- **训练单元**（`CFAR_TRAIN_CELLS = 8`）：在每个待检测bin的两侧各取 8 个bin作为噪声参考。
- **保护单元**（`CFAR_GUARD_CELLS = 2`）：在待检测bin和训练单元之间留 2 个bin的间隔。如果待检测bin是真实峰值，其能量会泄漏到相邻bin——保护单元防止这种泄漏污染噪声估计。
- **自适应阈值**（`CFAR_ALPHA * (noise_sum / noise_count)`）：阈值 = 4.0 × 局部噪声均值。这意味着每个频率bin的阈值都是根据其周围噪声水平动态计算的——在噪声高的频段阈值自动升高，在噪声低的频段自动降低。这比固定阈值方法在各种环境下都能保持一致的虚警率。
- **峰值条件**：`mag[i] > local_thr`（超过阈值）AND `mag[i] > mag[i-1]` AND `mag[i] > mag[i+1]`（局部极大值）。两个条件同时满足才算一个峰值，排除平顶噪声的误判。

---

### 9. 多普勒脉冲对自相关

```c
// STEP 7 — DOPPLER PATH: pulse-pair autocorrelation update
bool pair_ok = false;
if (last_proc_ts != 0) {
    float dt = (float)(packet->ts_us - last_proc_ts) * 1e-6f;
    if (dt > 0.02f && dt < PAIR_MAX_DT_S) {
        t_avg = 0.92f * t_avg + 0.08f * dt;
        pair_ok = true;
    }
}
last_proc_ts = packet->ts_us;

for (int s = 0; s < sc_count; s++) {
    if (!ok_slot[s]) {
        pp_prev_ok[s] = false;
        pp_R1_re[s] *= PP_LAMBDA;
        pp_R1_im[s] *= PP_LAMBDA;
        pp_R0[s]    *= PP_LAMBDA;
        continue;
    }

    float m  = mag_slot[s] * inv_mean_mag;
    float re = m * cosf(phi_slot[s]);
    float im = m * sinf(phi_slot[s]);

    if (pp_prev_ok[s] && pair_ok) {
        float p_re = re * pp_prev_re[s] + im * pp_prev_im[s];
        float p_im = im * pp_prev_re[s] - re * pp_prev_im[s];
        float pwr  = 0.5f * (re * re + im * im +
                             pp_prev_re[s] * pp_prev_re[s] +
                             pp_prev_im[s] * pp_prev_im[s]);

        pp_R1_re[s] = PP_LAMBDA * pp_R1_re[s] + p_re;
        pp_R1_im[s] = PP_LAMBDA * pp_R1_im[s] + p_im;
        pp_R0[s]    = PP_LAMBDA * pp_R0[s]    + pwr;
    }

    pp_prev_re[s] = re;
    pp_prev_im[s] = im;
    pp_prev_ok[s] = true;
}
```

**逐段解析：**

脉冲对（Pulse-Pair）算法是多普勒气象雷达中的经典方法，用于估计运动目标的平均径向速度：

- **时间间隔检查**（`dt > 0.02f && dt < PAIR_MAX_DT_S`）：只接受 20ms-200ms 之间的时间间隔。太短（<20ms）可能是重复包，太长（>200ms）的多普勒相位已经失相关。
- **复信号重构**（`re = m * cosf(phi)`, `im = m * sinf(phi)`）：将幅度和相位转换回复数表示。这是脉冲对算法的输入格式。
- **一阶自相关 R1**（`p_re = re * pp_prev_re + im * pp_prev_im`）：当前包与前一包的复共轭乘积。R1 的相位角正比于多普勒频移——人靠近时 R1 的相位为正，远离时为负。
- **零阶自相关 R0**（`pwr = 0.5 * (|z|^2 + |z_prev|^2)`）：信号功率，用作归一化基准。
- **EWMA 平滑**（`PP_LAMBDA = 0.97`）：指数加权移动平均，等效窗口约 33 个样本（1.6 秒 @ 20Hz）。这使得多普勒估计在时间上平滑，减少单次估计的方差。

---

### 10. 跨子载波多普勒聚合

```c
// STEP 8 — Cross-subcarrier Doppler aggregation
if (doppler_ready) {
    float Sw = 0.0f, Swf = 0.0f, Swf2 = 0.0f, Sww = 0.0f;
    float V_re = 0.0f, V_im = 0.0f, S_absR1 = 0.0f;
    int agg_count = 0;
    float two_pi_T = 2.0f * (float)M_PI * t_avg;

    for (int s = 0; s < sc_count; s++) {
        if (pp_R0[s] < 1e-6f) continue;

        float absR1 = sqrtf(pp_R1_re[s] * pp_R1_re[s] +
                            pp_R1_im[s] * pp_R1_im[s]);
        float rho = absR1 / pp_R0[s];
        if (rho > 1.0f) rho = 1.0f;

        float width = 1.0f - rho;
        float f_k = atan2f(pp_R1_im[s], pp_R1_re[s]) / two_pi_T;

        float w = pp_R0[s];
        Sw   += w;
        Swf  += w * f_k;
        Swf2 += w * f_k * f_k;
        Sww  += w * width;

        V_re    += pp_R1_re[s];
        V_im    += pp_R1_im[s];
        S_absR1 += absR1;
        agg_count++;
    }

    if (doppler_measured) {
        float F_mean = Swf / Sw;
        float F_var  = Swf2 / Sw - F_mean * F_mean;
        float D_now  = (F_var > 0.0f) ? sqrtf(F_var) : 0.0f;
        float W_now  = Sww / Sw;
        float A_now  = sqrtf(V_re * V_re + V_im * V_im) / (S_absR1 + 1e-9f);

        W_s = AGG_SMOOTH_ALPHA * W_now  + (1.0f - AGG_SMOOTH_ALPHA) * W_s;
        D_s = AGG_SMOOTH_ALPHA * D_now  + (1.0f - AGG_SMOOTH_ALPHA) * D_s;
        F_s = AGG_SMOOTH_ALPHA * F_mean + (1.0f - AGG_SMOOTH_ALPHA) * F_s;
        A_s = AGG_SMOOTH_ALPHA * A_now  + (1.0f - AGG_SMOOTH_ALPHA) * A_s;
    }
}
```

**逐段解析：**

将每个子载波独立的脉冲对结果聚合为四个全局多普勒特征：

- **频谱宽度 W**（`width = 1 - rho`）：`rho = |R1|/R0` 是归一化自相关系数。rho 接近 1 表示信号高度相关（静止环境），rho 接近 0 表示信号去相关（运动剧烈）。`1 - rho` 越大多普勒频谱越宽——人走动时 W 大，静坐时 W 小。
- **多普勒频移 F**（`atan2f(R1_im, R1_re) / (2πT)`）：R1 的相位角除以 2πT 得到多普勒频率。正值=接近，负值=远离。
- **频谱展宽 D**（`sqrt(F_var)`）：多普勒频率的加权标准差。多人时 D 大（不同人有不同速度），单人时 D 小。这个特征后来用于人数估计。
- **对齐度 A**（`|ΣR1| / Σ|R1|`）：所有子载波 R1 矢量和的模除以模的和。如果所有子载波的多普勒方向一致（单人），A 接近 1；如果方向不一致（多人或复杂运动），A 较低。
- **功率加权**（`w = pp_R0[s]`）：用每个子载波的信号功率作为权重，强信号子载波贡献更大，弱信号子载波贡献更小。

---

### 11. 状态机 FSM — 融合决策

```c
// STEP 10 — EVIDENCE SIGNALS (OR-entry / AND-exit fusion)

// --- Amplitude evidence ---
bool amp_entry     = (smooth_presence_metric > amp_entry_thr);
bool amp_below_exit = (smooth_presence_metric < amp_exit_thr);

// --- Doppler evidence ---
bool dopp_entry     = (W_s > dopp_w_enter) || (D_s > dopp_d_enter);
bool dopp_below_exit = (W_s < dopp_w_exit) && (D_s < dopp_d_exit);

// --- CFAR FFT evidence (needs corroboration from amplitude) ---
bool cfar_entry = (latest_cfar_peaks > 0) &&
                  (smooth_presence_metric > amp_noise_median + 2.0f * amp_noise_sigma);

// --- Breathing evidence (holds occupancy, does not trigger entry) ---
bool breathing_evidence = breathing_ready &&
                          (breathing_variance > breathing_threshold);

// --- Phase coherence evidence ---
bool phase_presence = (phase_coherence < phase_coherence_thr) &&
                      (phase_coherence > 0.05f);

// --- Fused entry (rate-adaptive) ---
bool doppler_functional = (doppler_health > DOPP_HEALTH_ENTER);
bool amp_only_entry = (smooth_presence_metric > amp_entry_thr * AMP_ONLY_ENTRY_MULT);
bool any_entry_evidence = doppler_functional
    ? ((amp_entry && dopp_entry) || cfar_entry)
    : amp_only_entry;

// --- Fused exit: ALL exit conditions met ---
bool all_exit_quiet = amp_below_exit && dopp_below_exit &&
                      !breathing_hold && !phase_hold;
```

**逐段解析：**

这是整个系统的决策核心——融合逻辑：

- **AND 进入逻辑**：当多普勒可用时（`doppler_health > 0.4`），需要幅度 AND 多普勒同时超过阈值才触发进入。双链路互相印证，大幅降低单链路的误报率。或者 CFAR FFT 检测到峰值且幅度超过 2σ 也可触发——这是对缓慢运动（如静坐呼吸）的补充检测路径。
- **低 CSI 速率回退**：当 CSI 速率太低（`doppler_health < 0.4`）时，多普勒路径不可用，回退到幅度-only 检测，但阈值加倍（`AMP_ONLY_ENTRY_MULT = 2.0`）。更严格的阈值避免幅度噪声导致的误报，同时静态信道逃逸机制保证误报会在几秒内自动清除。
- **呼吸/相位仅保持**：呼吸和相位相干性不能触发进入（从 EMPTY 状态），只能保持已占用的状态。这是因为 HVAC 气流、Wi-Fi 速率波动等环境因素也会产生类似呼吸的信号——如果允许它们触发进入，会导致空房间被误判为有人。
- **幅度门控保持**（`amp_supports_presence`）：呼吸和相位保持信号还需要幅度高于退出阈值才有效。一旦幅度回到空房间水平，残留的呼吸带波动或低相位相干性被视为环境噪声（HVAC、漂移等），而非人体。

```c
// STEP 11 — FINITE STATE MACHINE
switch (current_state) {
    case STATE_EMPTY:
        if (any_entry_evidence) {
            enter_confirm_sec += fsm_dt;
        } else {
            enter_confirm_sec = 0.0f;
        }
        if (enter_confirm_sec >= ENTER_CONFIRM_SEC) {
            current_state = is_walking ? STATE_OCCUPIED_MOVING
                                      : STATE_OCCUPIED_STATIONARY;
        }
        // Online recalibration during stable EMPTY...
        break;

    case STATE_OCCUPIED_MOVING:
    case STATE_OCCUPIED_STATIONARY: {
        // Leaky exit integrator in seconds (AND-exit)
        if (all_exit_quiet) {
            exit_quiet_sec += fsm_dt;
        } else {
            exit_quiet_sec -= EXIT_PENALTY_FACTOR * fsm_dt;
            if (exit_quiet_sec < 0.0f) exit_quiet_sec = 0.0f;
        }
        bool exit_now = (exit_quiet_sec >= EXIT_QUIET_SEC);

        // No-motion escape — the GUARANTEED return-to-EMPTY path
        bool static_channel = (time_variance < walking_threshold) &&
                              (W_s < dopp_w_enter) && (D_s < dopp_d_enter);
        if (static_channel) {
            no_motion_sec += fsm_dt;
        } else {
            no_motion_sec -= STATIC_DECAY_FACTOR * fsm_dt;
            if (no_motion_sec < 0.0f) no_motion_sec = 0.0f;
        }
        float static_limit = (smooth_presence_metric > amp_entry_thr)
            ? STATIC_ESCAPE_SEC * STATIC_OCCUPIED_FACTOR
            : STATIC_ESCAPE_SEC;
        if (no_motion_sec >= static_limit) {
            rebaseline_boost = REBASELINE_PACKETS;
            exit_now = true;
        }
        // ...
    }
}
```

**逐段解析：**

- **进入确认**（`enter_confirm_sec >= ENTER_CONFIRM_SEC`）：需要连续 0.5 秒的证据累积才从 EMPTY 转入 OCCUPIED。这过滤了瞬态干扰（如关门震动）。
- **泄漏积分器退出**（`exit_quiet_sec`）：不是简单的"连续 N 秒安静就退出"，而是一个泄漏积分器——安静时 `+Δt`，有活动时 `-2×Δt`（惩罚因子）。这意味着偶尔的活动不会重置退出计时器，但会显著减缓它。需要净 10 秒的安静才能确认退出。
- **静态信道逃逸**：这是最关键的防死锁机制。当信道完全静止（无运动、无多普勒）超过 15 秒（或强信号时 60 秒），强制重新基线化并返回 EMPTY。这解决了"人走后基线偏移导致幅度永远高于阈值"的死锁问题——人走后信道变为静态，而真正的人会持续扰动信道。
- **静态限制自适应**：如果幅度仍然很高（`> amp_entry_thr`），将等待时间延长 4 倍——这可能是静止不动的人，给他更多时间移动。但最终一定会触发（无死锁）。

---

### 12. NVS 校准持久化

```c
#define RADAR_CALIB_MAGIC    0x48594235   // "HYB5"

typedef struct {
    uint32_t magic;
    float amp_noise_median;
    float amp_noise_sigma;
    float amp_entry_threshold;
    float amp_exit_threshold;
    float dopp_width_floor;
    float dopp_width_enter;
    float dopp_width_exit;
    float dopp_spread_floor;
    float dopp_spread_enter;
    float dopp_spread_exit;
    float walking_threshold;
    float breathing_threshold;
    float phase_coherence_threshold;
} radar_calib_t;

static bool load_saved_calibration(radar_calib_t *out)
{
    nvs_handle_t h;
    if (nvs_open("radar", NVS_READONLY, &h) != ESP_OK) return false;
    size_t len = sizeof(*out);
    esp_err_t err = nvs_get_blob(h, "calib", out, &len);
    nvs_close(h);
    return (err == ESP_OK) && (len == sizeof(*out)) && (out->magic == RADAR_CALIB_MAGIC);
}
```

**逐段解析：**

- **Magic Number**（`0x48594235` = "HYB5"）：结构体布局的版本标识。每次修改 `radar_calib_t` 的字段就必须 bump 这个值，使旧的 NVS 数据自动失效——防止旧版本的数据被错误解析为新格式。
- **NVS Blob 存储**：整个校准结构体作为一个 blob 存储在 NVS（Non-Volatile Storage）中。NVS 是 ESP-IDF 提供的键值对 Flash 存储，适合小量配置数据。
- **三重验证**：加载时验证（1）NVS 打开成功，（2）blob 大小匹配，（3）magic number 匹配。任何一项不匹配都返回 false，使用默认阈值。

校准流程中的疑似检测逻辑：

```c
// NVS suspect check
radar_calib_t saved;
bool have_saved = load_saved_calibration(&saved);
if (have_saved &&
    amp_entry_thr > saved.amp_entry_threshold * CALIB_SUSPECT_RATIO) {
    ESP_LOGW(TAG, "Calibration looks contaminated...");
    // Use saved thresholds instead
    amp_entry_thr = saved.amp_entry_threshold;
    // ... restore all saved thresholds
} else {
    // Store fresh calibration
    store_calibration(&fresh);
}
```

- **疑似检测**：如果新校准的进入阈值是保存阈值的 2.5 倍以上（`CALIB_SUSPECT_RATIO = 2.5`），说明校准时房间可能有人（导致噪声估计偏高）。此时使用上次保存的良好阈值，而非被污染的新阈值。
- **正常存储**：如果新阈值合理，存入 NVS 供下次使用。

---

### 13. FFT 实现 — Radix-2 Cooley-Tukey

```c
// fft.c — In-place Radix-2 Cooley-Tukey FFT

static uint16_t reverse_bits(uint16_t val, int bits) {
    uint16_t res = 0;
    for (int i = 0; i < bits; i++) {
        res = (res << 1) | (val & 1);
        val >>= 1;
    }
    return res;
}

void fft_compute(float *real, float *imag, uint16_t n) {
    int levels = 0;
    uint16_t temp = n;
    while (temp > 1) { levels++; temp >>= 1; }

    // Bit-reversal permutation
    for (uint16_t i = 0; i < n; i++) {
        uint16_t j = reverse_bits(i, levels);
        if (j > i) {
            float t_real = real[i]; real[i] = real[j]; real[j] = t_real;
            float t_imag = imag[i]; imag[i] = imag[j]; imag[j] = t_imag;
        }
    }

    // Cooley-Tukey decimation-in-time
    for (uint16_t size = 2; size <= n; size *= 2) {
        uint16_t half_size = size / 2;
        float angle_step = -2.0f * M_PI / size;

        for (uint16_t i = 0; i < n; i += size) {
            for (uint16_t j = i; j < i + half_size; j++) {
                uint16_t k = j - i;
                float angle = k * angle_step;
                float twiddle_real = cosf(angle);
                float twiddle_imag = sinf(angle);
                float t_real = real[j + half_size] * twiddle_real
                             - imag[j + half_size] * twiddle_imag;
                float t_imag = real[j + half_size] * twiddle_imag
                             + imag[j + half_size] * twiddle_real;
                real[j + half_size] = real[j] - t_real;
                imag[j + half_size] = imag[j] - t_imag;
                real[j] += t_real;
                imag[j] += t_imag;
            }
        }
    }
}
```

**逐段解析：**

这是一个经典的就地（in-place）Radix-2 DIT（Decimation In Time）FFT 实现：

- **位反转排列**：FFT 要求输入按位反转顺序排列。`reverse_bits` 将索引的二进制位反转（如 001 → 100）。通过 `if (j > i)` 只交换一半，避免重复交换。
- **蝶形运算**：外层循环 `size` 从 2 到 N 逐级翻倍——每一级处理 2 点、4 点、8 点...直到 N 点 DFT。内层循环对每个蝶形组执行标准的 Cooley-Tukey 蝶形运算：
  - `twiddle`（旋转因子）：`e^(-2πik/size)` 的实部和虚部。
  - 蝶形：`X = A + W*B`, `Y = A - W*B`，其中 A 是前半部分，B 是后半部分。
- **就地计算**：所有操作在原数组上进行，不需要额外分配缓冲区——这对内存有限的 ESP32-C3 非常重要。
- **复数乘法**：`(a+bi)*(c+di) = (ac-bd) + (ad+bc)i`，代码中 `t_real = re*c - im*d`, `t_imag = re*d + im*c` 正是这个公式。

---

### 14. Wi-Fi 事件处理与流量生成

```c
static void wifi_event_handler(void *arg, esp_event_base_t event_base,
                               int32_t event_id, void *event_data)
{
    if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_START) {
        esp_wifi_connect();
    } else if (event_base == WIFI_EVENT && event_id == WIFI_EVENT_STA_DISCONNECTED) {
        ap_bssid_valid = false;
        esp_wifi_set_csi(false);
        xEventGroupClearBits(wifi_event_group, WIFI_CONNECTED_BIT);
        if (ping_handle) esp_ping_stop(ping_handle);
        wifi_reconnected = true;
        esp_wifi_connect();
    } else if (event_base == IP_EVENT && event_id == IP_EVENT_STA_GOT_IP) {
        ip_event_got_ip_t *event = (ip_event_got_ip_t *)event_data;
        dynamic_gw_ip.type = IPADDR_TYPE_V4;
        dynamic_gw_ip.u_addr.ip4.addr = event->ip_info.gw.addr;

        wifi_ap_record_t ap_info;
        if (esp_wifi_sta_get_ap_info(&ap_info) == ESP_OK) {
            memcpy(ap_bssid, ap_info.bssid, sizeof(ap_bssid));
            ap_bssid_valid = true;
        }

        esp_wifi_set_ps(WIFI_PS_NONE);
        initialize_native_ping_engine();
        esp_wifi_set_csi(true);
        xEventGroupSetBits(wifi_event_group, WIFI_CONNECTED_BIT);
    }
}
```

**逐段解析：**

- **断线处理**：设置 `wifi_reconnected = true` 标志。信号处理任务在下一个包到来时会检测到这个标志并执行完整重置——清空所有状态、重新校准。这是必要的，因为断线重连后 AP 可能切换了信道或天线，旧的基线和多普勒状态全部失效。
- **BSSID 锁定**：获取 IP 后立即记录 AP 的 BSSID。CSI 回调中用这个 BSSID 过滤——只接受来自关联 AP 的 CSI，拒绝其他 AP 的干扰包。
- **`WIFI_PS_NONE`**：关闭 Wi-Fi 省电模式。省电模式会引入不规则的睡眠间隔，导致 CSI 包间隔不稳定，破坏多普勒脉冲对的时间一致性。
- **双流量生成器**：Ping（25ms 间隔）+ DNS（50ms 间隔）。双路径的原因是某些路由器会限制 ICMP 速率或修改 CSI 质量，DNS 提供了一条备用激发路径。

---

### 15. DNS 流量生成任务

```c
static void dns_traffic_task(void *arg)
{
    static const uint8_t dns_query[] = {
        0xC5, 0x17, 0x00, 0x00, 0x00, 0x01,
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
        3, 'c', 's', 'i', 5, 'l', 'o', 'c', 'a', 'l', 0,
        0x00, 0x01, 0x00, 0x01,
    };

    while (1) {
        xEventGroupWaitBits(wifi_event_group, WIFI_CONNECTED_BIT,
                            pdFALSE, pdFALSE, portMAX_DELAY);

        int sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_IP);
        if (sock < 0) { vTaskDelay(pdMS_TO_TICKS(1000)); continue; }
        fcntl(sock, F_SETFL, O_NONBLOCK);

        struct sockaddr_in dest = {0};
        dest.sin_family = AF_INET;
        dest.sin_port = htons(53);

        while (xEventGroupGetBits(wifi_event_group) & WIFI_CONNECTED_BIT) {
            dest.sin_addr.s_addr = dynamic_gw_ip.u_addr.ip4.addr;
            if (dest.sin_addr.s_addr != 0) {
                sendto(sock, dns_query, sizeof(dns_query), 0,
                       (struct sockaddr *)&dest, sizeof(dest));
                uint8_t rx[128];
                while (recv(sock, rx, sizeof(rx), 0) > 0) {}
            }
            vTaskDelay(pdMS_TO_TICKS(50));
        }
        close(sock);
    }
}
```

**逐段解析：**

- **预构造 DNS 查询**：一个对 `csi.local` 的 A 记录查询。查询是手工构造的字节数组，避免了 DNS 解析库的开销。`0xC5, 0x17` 是随机事务 ID，`0x00, 0x01` 表示标准查询。
- **非阻塞 socket**（`O_NONBLOCK`）：`recv` 不会阻塞——如果路由器响应了 DNS 回复，快速读取并丢弃；如果没有响应，立即返回。这保证了流量生成任务的周期性不被 DNS 回复延迟。
- **网关目标**：DNS 查询发送到网关 IP（路由器）。这确保 Wi-Fi 帧在 ESP32 和路由器之间传输——正是 CSI 提取所需要的。发送到外部 DNS 服务器可能导致流量不经过路由器的 Wi-Fi 接口。
- **50ms 间隔**：与 Ping 的 25ms 间隔叠加，理论上可产生 60Hz 的 Wi-Fi 帧。但 CSI 回调中的限速器将实际处理速率控制在 ~20Hz—— oversampling 确保即使某些包被丢弃或延迟，仍有足够的 CSI 数据。

---

### 16. Python 实时绘图工具

```python
#!/usr/bin/env python3
"""Live presence plotter for the Hybrid CSI Presence Radar (ESP32-C3)."""

def reader():
    """Background thread: parse PLOT lines into the rolling buffers."""
    while True:
        line = ser.readline().decode("ascii", errors="ignore").strip()
        if not line.startswith("PLOT,"):
            continue
        parts = line.split(",")
        if len(parts) != 6:
            continue
        try:
            r, s, b, t = (float(parts[1]), float(parts[2]),
                          float(parts[3]), float(parts[4]))
            o = int(parts[5])
        except ValueError:
            continue
        with lock:
            raw.append(r); smooth.append(s); breath.append(b)
            thr.append(t); occ.append(o)

# ...
def update(_frame):
    with lock:
        r, s, b, t, o = (list(raw), list(smooth), list(breath),
                         list(thr), list(occ))
    l_raw.set_ydata(r)
    l_smooth.set_ydata(s)
    l_thr.set_ydata(t)
    l_breath.set_ydata(b)
    # Shade the whole background by the most recent occupancy state.
    occ_span[0].remove()
    color = "red" if (o and o[-1]) else "green"
    occ_span[0] = ax1.axvspan(0, N, color=color, alpha=0.08)
    return l_raw, l_smooth, l_thr, l_breath

_anim = animation.FuncAnimation(fig, update, interval=100, blit=False,
                                cache_frame_data=False)
```

**逐段解析：**

- **双线程架构**：后台线程持续读取串口数据并解析 `PLOT,raw,smooth,breath,threshold,occupied` 格式的遥测行；主线程通过 `FuncAnimation` 每 100ms 更新一次图表。
- **线程锁**（`with lock`）：保护共享的 deque 缓冲区。串口读取线程写入，动画回调线程读取，锁防止数据竞争。
- **背景着色**：根据最新占用状态将图表背景着色为绿色（EMPTY）或红色（OCCUPIED），直观展示状态转换时刻。
- **双面板**：上面板显示原始/平滑存在度量及进入阈值线；下面板显示呼吸带信号——可以直接看到静止呼吸的人在呼吸带上的周期性振荡。

---

## API 使用指南

### 串口输出格式

固件每秒输出一行状态信息：

```
ROOM: OCCUPIED  | ACT: BREATHING  | DIR: STATIC   | N=1 | LVL: LOW    | CONF: 72%
  AMP p=0.0342 in=0.0500 out=0.0120 | DOPP W=0.0123/0.0080 sF=0.0456/0.0300 | A=0.78
  exit 12/300 | breath=0.000045/0.000030 | phase_coh=0.520/0.650 | cfar=0 | still=0
  CSI rx 45.2/s our-AP 22.1/s accept 19.8/s foreign 23.1/s
```

| 字段 | 含义 |
|------|------|
| ROOM | EMPTY / OCCUPIED |
| ACT | NONE / MOVING / BREATHING / SITTING / STATIONARY |
| DIR | STATIC / APPROACHING / RECEDING / CROSSING |
| N | 估计人数 (1-4) |
| LVL | NONE / LOW / MEDIUM / HIGH |
| CONF | 置信度 (1-99%) |
| AMP p/in/out | 当前幅度 / 进入阈值 / 退出阈值 |
| DOPP W/enter | 频谱宽度 / 进入阈值 |
| sF/enter | 频谱展宽 / 进入阈值 |
| exit | 退出计时器 / 目标秒数 |
| breath | 呼吸方差 / 阈值 |
| phase_coh | 相位相干性 / 阈值 |
| cfar | CFAR 检测到的峰值数 |

### GPIO 接口

```python
# Python 示例：通过串口读取占用状态
import serial

ser = serial.Serial('/dev/ttyUSB0', 115200, timeout=1)
while True:
    line = ser.readline().decode().strip()
    if line.startswith('ROOM:'):
        occupied = 'OCCUPIED' in line
        print(f"Room: {'有人' if occupied else '无人'}")
```

```bash
# cURL 示例：通过 GPIO Web 服务器读取（需额外部署）
curl http://esp32-c3.local/gpio
# 返回: {"gpio": 4, "level": 1, "occupied": true}
```

### 实时绘图

```bash
pip install pyserial matplotlib
python plot_presence.py --port /dev/ttyUSB0
# 或 Windows:
python plot_presence.py --port COM5
```

---

## 编译与部署

### 环境搭建

| 工具 | 版本要求 | 说明 |
|------|----------|------|
| ESP-IDF | v5.4+ | Espressif IoT Development Framework |
| Python | 3.8+ | 用于 idf.py 工具链 |
| CMake | 3.16+ | 构建系统 |
| USB 驱动 | — | ESP32-C3 的 USB-UART 驱动 |

### 编译配置

```bash
# 1. 设置目标芯片
idf.py set-target esp32c3

# 2. 配置 Wi-Fi 和 GPIO
idf.py menuconfig
# → AC Presence Module Configuration
#   → WiFi SSID:        你的路由器 SSID
#   → WiFi Password:    你的路由器密码
#   → Occupancy output GPIO number: 4 (或 -1 禁用)
```

| menuconfig 选项 | 默认值 | 说明 |
|-----------------|--------|------|
| WiFi SSID | "IIC2023" | 路由器网络名称 |
| WiFi Password | — | 路由器密码 |
| PRESENCE_GPIO_NUM | 4 | 占用输出 GPIO（-1 禁用） |

### 烧录步骤

```bash
# 3. 编译
idf.py build

# 4. 烧录并监控
idf.py build flash monitor

# 5. 观察输出
# 首次启动约 15 秒后完成校准
# 看到 "=== CALIBRATION COMPLETE (HYBRID) ===" 表示就绪
```

### 调参指南

| 参数 | 默认值 | 调节效果 |
|------|:------:|----------|
| `EXIT_QUIET_SEC` | 10.0 | 增大=更慢退出（减少误清空），减小=更快退出 |
| `CALIB_K_PRESENCE` | 4.5 | 增大=更难进入（减少误报），减小=更敏感 |
| `STATIC_ESCAPE_SEC` | 15.0 | 增大=更容忍静止不动的人，减小=更快清空 |
| `ENTER_CONFIRM_SEC` | 0.5 | 增大=更难进入，减小=更快响应 |
| `PP_LAMBDA` | 0.97 | 增大=多普勒更平滑但响应慢，减小=更灵敏但噪声大 |

---

## 项目亮点与适用场景

### 项目亮点

1. **零传感器人体检测**：仅靠 Wi-Fi CSI 信号实现，无需 PIR、摄像头等额外传感器，硬件成本极低
2. **双链路融合架构**：幅度 + 多普勒双路径互相印证，从根本上解决单路径误报问题
3. **呼吸级灵敏度**：通过双 EWMA 带通和相位相干性指数，可以检测到静坐呼吸的人——这是 PIR 传感器做不到的
4. **生产级防死锁设计**：静态信道逃逸机制保证了系统永远不会卡在"有人"状态，即使基线漂移
5. **自适应校准**：NVS 持久化 + 疑似检测 + 在线重校准，适应各种环境变化
6. **完整的开发工具链**：含 Python 实时绘图器，方便调试和可视化

### 适用场景

| 场景 | 说明 |
|------|------|
| 智能 AC 控制 | 嵌入空调，无人自动关机，节能 |
| 智能家居占用检测 | 房间有人/无人状态，联动灯光、新风 |
| 安防监控 | 无摄像头隐私泄露风险的入侵检测 |
| 老人看护 | 呼吸检测判断是否在房间内活动 |
| 办公室占用统计 | 人数统计 + 占用率分析 |
| 酒店节能 | 客人离开后自动关闭空调和灯光 |

---

## 总结

`wifi-presence-detector` 是一个技术含量极高的 ESP32-C3 固件项目，展示了如何仅用 Wi-Fi CSI 信号实现生产级人体存在检测。项目最令人印象深刻的是其**双链路融合架构**——幅度路径和多普勒路径各自独立处理，再通过精心设计的 AND 进入/AND 退出融合逻辑互相印证，大幅降低了误报率。

从代码质量角度看，项目展现了深厚的信号处理功底：
- 脉冲对自相关算法（气象雷达级技术）被巧妙地移植到 Wi-Fi CSI 领域
- CFAR 自适应阈值检测使得系统在不同噪声环境下都能保持一致的虚警率
- 双 EWMA 带通滤波器用极低计算成本实现了呼吸频带隔离
- 静态信道逃逸机制和 NVS 疑似检测体现了丰富的生产环境经验

这个项目的意义远超空调控制——它证明了一个不到 $5 的 ESP32-C3 开发板就能实现过去需要专用雷达硬件才能做到的人体存在检测，为智能家居和 IoT 领域提供了一个极具性价比的方案。

---

> 📝 作者：蔡浩宇（jun-chy）  
> 📅 日期：2026-07-13  
> 🔗 项目地址：[https://github.com/KavyaSinghal05/wifi-presence-detector](https://github.com/KavyaSinghal05/wifi-presence-detector)
