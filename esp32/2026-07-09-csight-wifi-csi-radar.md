# CSIght 深度解析：用 WiFi 信道状态信息实现穿墙运动检测的 Flipper Zero + ESP32 雷达

> 📅 日期：2026-07-09  
> 🏷️ 分类：ESP32 / Flipper Zero  
> ⭐ Star 数：142  
> ✍️ 作者：蔡浩宇（jun-chy）

---

## 项目链接

| 项目信息 | 详情 |
|---|---|
| **GitHub 地址** | https://github.com/joelewis012/CSIght |
| **作者** | joelewis012 |
| **许可证** | MIT |
| **最近更新** | 2026-06-19 |
| **主要语言** | C（99.1%） |
| **创建时间** | 2026-06-12 |
| **最新版本** | v1.1 |

---

## 一、项目简介

**CSIght** 是一个"看穿墙壁"的 WiFi 雷达系统——无需摄像头、红外或特殊传感器，仅凭 ESP32 板载的 WiFi 射频，就能实时检测隔墙的运动、存在与距离，并在 Flipper Zero 上渲染为动态雷达画面。

其原理是 **WiFi 信道状态信息（CSI, Channel State Information）**：每个穿越空间的 WiFi 数据包都携带该空间的环境特征——信号在数十个频率子载波上的幅度、相位与传播时延。当有人或物体移动时，这些数值会发生偏移。CSIght 实时读取这些偏移，通过基线校准与自适应滤波算法提取运动信号，最终转化为可视化雷达图像。

整个系统是**双设备协同架构**：ESP32 负责 WiFi CSI 采集与运动检测算法，Flipper Zero 负责显示与人机交互，两者通过 UART 协议通信。

### 核心特性一览

| 特性 | 说明 |
|---|---|
| **WiFi CSI 穿墙感知** | 利用 WiFi 子载波幅度变化检测运动，无需摄像头/红外 |
| **自动芯片检测** | ESP32 上电后主动上报芯片型号与 CSI 支持等级 |
| **50+ 开发板预设** | 覆盖官方 Flipper 板、DevKit、XIAO、Adafruit、SparkFun、M5Stack、LOLIN 等 |
| **三种显示模式** | 雷达扫描、瀑布图、近场距离，实时切换 |
| **灵敏度可调** | 0-10 级灵敏度，运行时调整检测阈值 |
| **距离估算** | 基于信号偏移强度粗略估算运动目标距离 |
| **自适应基线** | EMA 指数移动平均缓慢跟踪环境变化，抗误报 |
| **配置持久化** | 引脚配置与灵敏度存入 Flipper SD 卡，设一次永久生效 |
| **多芯片 CSI 适配** | 针对 ESP32-C6（WiFi 6）等不同芯片族使用不同 CSI 配置结构体 |

---

## 二、硬件架构

### 系统组成图

```
    ┌──────────────────────┐         UART 115200          ┌──────────────────────┐
    │      ESP32 板卡       │   TX(GPIO17) ──────────────► │    Flipper Zero      │
    │  (CSI 采集 + 算法)    │   RX(GPIO16) ◄────────────── │   (显示 + 交互)       │
    │                      │   GND ────────────────────── │                      │
    │  ┌────────────────┐  │   3.3V ◄──────────────────── │  ┌────────────────┐  │
    │  │  WiFi 射频      │  │                              │  │  128×64 LCD    │  │
    │  │  (被动嗅探)     │  │   协议帧:                    │  │  雷达/瀑布/距离 │  │
    │  │  Channel 6     │  │   0xAA 握手请求               │  └────────────────┘  │
    │  └───────┬────────┘  │   0xBB 握手应答(芯片信息)     │  ┌────────────────┐  │
    │          │ CSI回调    │   0x01 运动事件               │  │  按键:         │  │
    │          ▼            │   0x03 瀑布数据               │  │  ◄► 切换模式   │  │
    │  ┌────────────────┐  │   0x10 灵敏度设置             │  │  ▲▼ 调灵敏度   │  │
    │  │  csi_handler   │  │   0x11 模式设置               │  │  OK 确认       │  │
    │  │  · 基线校准     │  │                              │  └────────────────┘  │
    │  │  · 自适应滤波   │  │                              │                      │
    │  │  · 运动检测     │  │                              │  SD卡: 配置持久化     │
    │  │  · 距离估算     │  │                              │  (apps_data/csight)  │
    │  └────────────────┘  │                              │                      │
    └──────────────────────┘                              └──────────────────────┘
              │
              ▼
     WiFi 数据包穿越空间
     ┌───────────────────────────┐
     │     墙壁 / 空间 / 人体      │
     │  子载波幅度/相位随运动变化   │
     └───────────────────────────┘
```

### 接线配置

| ESP32 引脚 | Flipper GPIO | 说明 |
|---|---|---|
| TX（默认 GPIO 17） | Pin 14（RX） | ESP32 → Flipper 数据 |
| RX（默认 GPIO 16） | Pin 13（TX） | Flipper → ESP32 命令 |
| GND | GND | 共地 |
| 3.3V | 3.3V | 供电（须先给 ESP32 上电再启动 App） |

> 引脚可自定义，不同开发板预设不同 TX/RX 组合，配置存入 SD 卡。

### 硬件清单

| 器件 | 规格 | 用途 |
|---|---|---|
| ESP32 开发板 | ESP32 / S3 / C3 / C6 任一（推荐 S3 或 C6） | WiFi CSI 采集与运动算法 |
| Flipper Zero | 多功能安全工具 | 显示雷达界面与人机交互 |
| 杜邦线 ×4 | TX/RX/GND/3V3 | 双设备 UART 通信 |

### CSI 支持等级

| 芯片 | CSI 支持 | 说明 |
|---|---|---|
| ESP32-C6 | ✅ 完整 | WiFi 6，灵敏度最佳 |
| ESP32-C3 | ✅ 完整 | 高性价比方案 |
| ESP32-S3 | ✅ 完整 | 推荐全能型 |
| ESP32（原版） | ⚠️ 有限 | 仅幅度，运动检测可用，距离精度差 |
| ESP32-S2 | ❌ 无 | 不支持 CSI |
| ESP32-H2 | ❌ 无 | 仅 802.15.4，无 WiFi |

---

## 三、固件架构

### 文件结构

| 路径 | 大小 | 职责 |
|---|---|---|
| `esp32/main/main.c` | 7.6KB | ESP32 入口：UART 初始化、WiFi 配置、握手协议、命令处理 |
| `esp32/components/csi_handler/csi_handler.c` | 7.1KB | **核心算法：CSI 回调、基线校准、运动检测、距离估算** |
| `esp32/components/csi_handler/csi_handler.h` | 475B | 回调函数类型定义 |
| `esp32/sdkconfig.defaults` | 567B | ESP-IDF 配置 |
| `flipper/src/csight.h` | 5.9KB | App 状态结构体、协议常量、板卡预设定义 |
| `flipper/src/csight_app.c` | 22.2KB | Flipper 主逻辑：状态机、输入处理、板卡预设、配置 I/O、雷达光点 |
| `flipper/src/csight_uart.c` | 6.9KB | Flipper UART 收发：协议解析、RX 线程 |
| `flipper/src/csight_draw.c` | 16.8KB | 三种显示模式渲染 |

### 模块职责与数据流

```
感知层    │  WiFi 射频 (被动嗅探 Channel 6)
──────────┼─────────────────────────────────────────────
算法层    │  csi_handler.c
(ESP32)   │  ┌─────────┐   ┌───────────┐   ┌──────────┐
          │  │基线校准  │──►│自适应滤波  │──►│运动检测   │
          │  │(30帧均值)│   │(EMA α=0.02)│   │(阈值比较) │
          │  └─────────┘   └───────────┘   └────┬─────┘
          │                                     │
          │              ┌──────────────────────┤
          │              ▼                      ▼
          │      ┌──────────────┐      ┌──────────────┐
          │      │ 距离估算      │      │ 瀑布压缩     │
          │      │(delta→0-100%)│      │(64→32 bins)  │
          │      └──────┬───────┘      └──────┬───────┘
──────────┼─────────────┼─────────────────────┼──────────
通信层    │         运动事件帧 0x01       瀑布帧 0x03
(UART)    │              ◄──── UART 115200 ────►
──────────┼─────────────────────────────────────────────
显示层    │  csight_uart.c (协议解析) ──► csight_app.c (状态机)
(Flipper) │                                    │
          │              ┌─────────────────────┤
          │              ▼                     ▼
          │      ┌──────────────┐     ┌──────────────┐
          │      │ 雷达光点计算  │     │ 瀑布缓冲滚动  │
          │      │(COS64/SIN64) │     │(80列×10行)   │
          │      └──────┬───────┘     └──────┬───────┘
          │             ▼                    ▼
          │      csight_draw.c (128×64 LCD 渲染)
```

---

## 四、核心代码深度分析

> 以下代码均取自项目真实源码，逐段解析其设计意图与实现细节。

### 4.1 ESP32 WiFi 被动 CSI 嗅探—— `main.c`

ESP32 固件的核心任务是：初始化 WiFi 为 STA 模式并锁定信道，被动嗅探该信道上所有 WiFi 数据包的 CSI，然后通过 UART 与 Flipper 通信。

```c
// ─── 通信协议常量 ───────────────────────────────────────────
#define PROTO_HANDSHAKE_REQ  0xAA
#define PROTO_HANDSHAKE_ACK  0xBB
#define PROTO_MOTION_EVT     0x01
#define PROTO_WATERFALL_EVT  0x03
#define PROTO_SENSITIVITY    0x10
#define PROTO_MODE_SET       0x11

// ─── WiFi 初始化（STA 模式，被动 CSI 嗅探）────────────────
static void wifi_init(void) {
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
    ESP_ERROR_CHECK(esp_wifi_set_storage(WIFI_STORAGE_RAM));
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    ESP_ERROR_CHECK(esp_wifi_start());
    ESP_ERROR_CHECK(esp_wifi_set_channel(6, WIFI_SECOND_CHAN_NONE));
    ESP_LOGI(TAG, "WiFi started, channel 6");
}
```

**逐段解析：**

- 协议设计简洁：`0xAA`/`0xBB` 握手、`0x01` 运动事件、`0x03` 瀑布数据、`0x10`/`0x11` 配置命令。单字节命令头便于 Flipper 端流式解析。
- `wifi_init` 是整个感知能力的基石。关键在于 **不连接任何 AP**——仅 `set_mode(STA)` + `start()` + `set_channel(6)`。ESP32 进入 STA 模式后，WiFi 射频会被动接收该信道上所有数据包，CSI 回调即可捕获环境中的 WiFi 信号变化。锁定信道 6（2.4GHz 最常用信道）保证持续监听同一频段。
- `WIFI_STORAGE_RAM` 把 WiFi 配置存入 RAM 而非 Flash，减少 Flash 磨损并加快启动。

### 4.2 自动芯片检测与握手协议—— `main.c`

ESP32 上电后等待 Flipper 发来 `0xAA` 握手请求，收到后回复包含芯片型号与 CSI 支持等级的应答帧：

```c
// 芯片型号映射
static const char* get_chip_name(esp_chip_model_t model) {
    switch(model) {
        case CHIP_ESP32:   return "ESP32";
        case CHIP_ESP32S3: return "ESP32-S3";
        case CHIP_ESP32C3: return "ESP32-C3";
        case CHIP_ESP32C6: return "ESP32-C6";
        // ...
    }
}

// CSI 支持等级：0=无, 1=有限(仅幅度), 2=完整(幅度+相位)
static uint8_t get_csi_support(esp_chip_model_t model) {
    switch(model) {
        case CHIP_ESP32:   return 1; // 有限 — 旧版 CSI API
        case CHIP_ESP32S3: return 2;
        case CHIP_ESP32C3: return 2;
        case CHIP_ESP32C6: return 2;
        case CHIP_ESP32S2: return 0; // 无 CSI
        case CHIP_ESP32H2: return 0; // 仅 802.15.4
        default:           return 1;
    }
}

// 握手应答帧构建: [0xBB][payload_len][chip_name\0][csi_support][fw_major][fw_minor][xor_chk]
static int build_handshake(uint8_t *buf, size_t buf_size) {
    esp_chip_info_t chip;
    esp_chip_info(&chip);

    const char *name    = get_chip_name(chip.model);
    uint8_t     support = get_csi_support(chip.model);

    size_t name_len     = strlen(name);
    uint8_t payload_len = (uint8_t)(name_len + 1 + 3); // name+null+support+maj+min

    int i = 0;
    buf[i++] = PROTO_HANDSHAKE_ACK;       // 帧头 0xBB
    buf[i++] = payload_len;                // 载荷长度
    memcpy(&buf[i], name, name_len + 1);   // 芯片名(含\0)
    i += (int)name_len + 1;
    buf[i++] = support;                    // CSI 支持等级
    buf[i++] = 1;                          // 固件主版本
    buf[i++] = 0;                          // 固件次版本

    uint8_t chk = 0;
    for(int j = 1; j < i; j++) chk ^= buf[j];  // XOR 校验
    buf[i++] = chk;

    return i;
}
```

**逐段解析：**

- `get_csi_support` 返回三档等级：`0`（S2/H2 无 CSI）、`1`（原版 ESP32 有限，仅幅度）、`2`（S3/C3/C6 完整，幅度+相位）。Flipper 据此决定是否允许进入扫描模式及显示兼容性警告。
- `build_handshake` 构建 TLV 式（Type-Length-Value）帧：帧头 `0xBB` + 载荷长度 + 芯片名（以 `\0` 结尾）+ 3 字节元数据 + XOR 校验。XOR 校验覆盖从长度字段到末尾的所有字节，可检测传输错误。
- 这种"设备自描述"设计让 CSIght 支持任意 ESP32 变种——Flipper 不需要预先知道接的是什么板子，握手后自动获取芯片能力。

### 4.3 运动事件与瀑布数据的 UART 上报—— `main.c`

当 CSI 算法检测到运动时，通过回调函数把结果打包成协议帧发送给 Flipper：

```c
// 运动事件帧: [0x01][intensity][proximity][xor]
void csight_on_motion(uint8_t intensity, uint8_t proximity) {
    uint8_t pkt[4];
    pkt[0] = PROTO_MOTION_EVT;     // 0x01
    pkt[1] = intensity;             // 运动强度 0-255
    pkt[2] = proximity;             // 距离百分比 0-100
    pkt[3] = pkt[0] ^ pkt[1] ^ pkt[2];  // XOR 校验
    uart_write_bytes(UART_PORT, (const char*)pkt, sizeof(pkt));
}

// 瀑布数据帧: [0x03][count][bin0..binN][xor]
void csight_on_waterfall(uint8_t *bins, uint8_t count) {
    uint8_t pkt[64];
    if(count > 60) count = 60;       // 限制最大 60 个 bin
    pkt[0] = PROTO_WATERFALL_EVT;   // 0x03
    pkt[1] = count;
    memcpy(&pkt[2], bins, count);
    uint8_t chk = 0;
    for(int i = 0; i < count + 2; i++) chk ^= pkt[i];
    pkt[count + 2] = chk;
    uart_write_bytes(UART_PORT, (const char*)pkt, count + 3);
}
```

**解析：** 运动事件仅 4 字节，极省带宽；瀑布帧把 32 个子载波压缩为单字节幅度（0-255），整帧约 35 字节。XOR 校验覆盖帧头到数据末尾。`count > 60` 的截断保护防止缓冲区溢出——这是嵌入式协议设计的必备防御。

### 4.4 CSI 核心算法——基线校准与运动检测 `csi_handler.c`

这是整个项目的技术核心。CSI 回调函数在每个 WiFi 数据包到达时被触发，执行"基线建立 → 偏移计算 → 自适应更新 → 运动判定"的完整流水线：

```c
// ─── 状态变量 ───────────────────────────────────────────────
#define CSI_BUF_LEN      64       // 最大子载波数
#define BASELINE_FRAMES  30       // 基线校准帧数

static float s_baseline[CSI_BUF_LEN];  // 环境基线（安静状态下的平均幅度）
static int   s_frame_count   = 0;
static bool  s_baseline_done = false;

// 灵敏度转阈值: sens 0 → 8.0(迟钝), sens 10 → 1.0(灵敏)
static float sensitivity_to_threshold(uint8_t sens) {
    return 8.0f - ((float)sens * 0.7f);
}

// 子载波复数幅度: sqrt(real² + imag²)
static float csi_amplitude(int8_t real, int8_t imag) {
    return sqrtf((float)(real * real) + (float)(imag * imag));
}

// 距离估算: 信号偏移越大 → 目标越近 → proximity 越高
static uint8_t delta_to_proximity(float avg_delta) {
    if(avg_delta < 0.0f)  avg_delta = 0.0f;
    if(avg_delta > 15.0f) avg_delta = 15.0f;
    return (uint8_t)((avg_delta / 15.0f) * 100.0f);
}
```

**解析：**

- `s_baseline` 存储每个子载波的"安静基线"——即无人移动时的平均幅度。WiFi CSI 中每个子载波对应一个复数（实部+虚部），`csi_amplitude` 用 `sqrt(real²+imag²)` 计算其模长（幅度）。
- `sensitivity_to_threshold` 把 0-10 的灵敏度映射为检测阈值：灵敏度越高（10），阈值越低（1.0），越容易触发；灵敏度越低（0），阈值越高（8.0），越不容易误报。
- `delta_to_proximity` 把平均偏移量映射为 0-100% 的距离百分比：偏移越大说明运动对信号干扰越强，目标越近。

核心回调函数：

```c
static void wifi_csi_cb(void *ctx, wifi_csi_info_t *info) {
    if(!info || !info->buf) return;

    int8_t *raw   = info->buf;              // 原始 CSI 数据: 交错存放 real/imag
    int     n_sub = info->len / 2;          // 子载波数 = 数据长度/2
    if(n_sub > CSI_BUF_LEN) n_sub = CSI_BUF_LEN;
    if(n_sub <= 0) return;

    // 1. 计算当前帧每个子载波的幅度
    float amp[CSI_BUF_LEN];
    memset(amp, 0, sizeof(amp));
    for(int i = 0; i < n_sub; i++) {
        amp[i] = csi_amplitude(raw[i * 2], raw[i * 2 + 1]);
    }

    // 2. 阶段一: 累积基线（前 30 帧求均值）
    if(!s_baseline_done) {
        for(int i = 0; i < n_sub; i++) s_baseline[i] += amp[i];
        s_frame_count++;
        if(s_frame_count >= BASELINE_FRAMES) {
            for(int i = 0; i < n_sub; i++)
                s_baseline[i] /= (float)BASELINE_FRAMES;
            s_baseline_done = true;
        }
        return;  // 基线建立期间不检测运动
    }

    // 3. 阶段二: 计算每个子载波相对基线的偏移
    float wf_bins[CSI_BUF_LEN];
    float total_delta = 0.0f;
    for(int i = 0; i < n_sub; i++) {
        float d = fabsf(amp[i] - s_baseline[i]);
        wf_bins[i]  = d;
        total_delta += d;
    }
    float avg_delta = total_delta / (float)n_sub;

    // 4. 自适应基线更新（EMA 指数移动平均）
    const float alpha = 0.02f;
    for(int i = 0; i < n_sub; i++) {
        s_baseline[i] = s_baseline[i] * (1.0f - alpha) + amp[i] * alpha;
    }

    float threshold = sensitivity_to_threshold(s_sensitivity);

    // 5. 运动判定
    switch(s_mode) {
        case 0: // 运动模式
        case 2: // 距离模式（同样检测运动，区别在 Flipper 端展示）
        {
            if(avg_delta > threshold) {
                uint8_t intensity = float_to_byte(avg_delta, 20.0f);
                uint8_t proximity = delta_to_proximity(avg_delta);
                if(s_motion_cb) s_motion_cb(intensity, proximity);
            }
            break;
        }
        case 1: // 瀑布模式
        {
            // 把 n_sub 个子载波压缩为 32 列
            #define WF_COLS 32
            uint8_t out[WF_COLS];
            int bins_per_col = n_sub / WF_COLS;
            if(bins_per_col < 1) bins_per_col = 1;
            for(int c = 0; c < WF_COLS; c++) {
                float sum = 0.0f;
                for(int b = 0; b < bins_per_col; b++) {
                    int idx = c * bins_per_col + b;
                    if(idx < n_sub) sum += wf_bins[idx];
                }
                out[c] = float_to_byte(sum / (float)bins_per_col, 20.0f);
            }
            if(s_waterfall_cb) s_waterfall_cb(out, WF_COLS);
            break;
        }
    }
}
```

**逐段深度解析：**

1. **原始 CSI 数据解析**：`info->buf` 是 `int8_t` 数组，子载波的实部和虚部交错存放（`raw[0]`=real₀, `raw[1]`=imag₀, `raw[2]`=real₁...）。因此子载波数 `n_sub = len / 2`。

2. **基线校准（阶段一）**：前 30 帧不检测运动，而是累加每个子载波的幅度。30 帧后求平均，得到"安静环境基线"。这一步至关重要——它建立了"无人移动时 CSI 应该是什么样"的参考。`BASELINE_FRAMES = 30` 平衡了校准速度与准确性。

3. **偏移计算（阶段二）**：对每个子载波计算 `|amp - baseline|`（绝对差值），再求所有子载波的平均偏移 `avg_delta`。当有人移动时，WiFi 多径效应改变，多个子载波的幅度同时偏移，`avg_delta` 显著增大。

4. **自适应基线更新（EMA）**：这是防误报的关键。`alpha = 0.02` 意味着基线每帧只吸收 2% 的新数据。这样，缓慢的环境变化（如温度漂移、家具微移）会被基线逐渐"吸收"而不触发运动；但突发的人体移动变化太快，基线来不及跟上，仍会被检测为偏移。公式 `baseline = baseline × 0.98 + amp × 0.02` 是经典的一阶 IIR 低通滤波器。

5. **运动判定**：`avg_delta > threshold` 即触发运动事件。阈值由灵敏度决定。运动强度 `intensity` 把 `avg_delta` 映射到 0-255（max=20.0），距离 `proximity` 映射到 0-100%。

6. **瀑布压缩**：把 64 个子载波的偏移值压缩为 32 列，每列是若干子载波的平均，便于 Flipper 的窄屏横向滚动显示。

### 4.5 多芯片 CSI 配置适配—— `csi_handler.c`

不同 ESP32 芯片族的 CSI 配置 API 不同，项目用条件编译适配：

```c
void csi_handler_init(csi_motion_cb_t motion_cb, csi_waterfall_cb_t waterfall_cb) {
    s_motion_cb    = motion_cb;
    s_waterfall_cb = waterfall_cb;

    memset(s_baseline, 0, sizeof(s_baseline));
    s_frame_count   = 0;
    s_baseline_done = false;

    // ── ESP32-C6/C61 使用 wifi_csi_acquire_config_t（WiFi 6 新结构体）──
#if CONFIG_IDF_TARGET_ESP32C6 || CONFIG_IDF_TARGET_ESP32C61
    wifi_csi_acquire_config_t csi_cfg = {
        .enable              = 1,
        .acquire_csi_legacy  = 1,   // L-LTF（传统前导码）
        .acquire_csi_ht20    = 1,   // HT20 数据包
        .acquire_csi_ht40    = 0,   // 禁用 HT40 以保证缓冲区大小可预测
        .acquire_csi_su      = 0,
        .acquire_csi_mu      = 0,
        .acquire_csi_dcm     = 0,
        .acquire_csi_beamformed = 0,
        .acquire_csi_he_stbc = 0,
        .val_scale_cfg       = 0,
        .dump_ack_en         = 0,
    };
#else
    // ── ESP32/S2/S3/C3 使用 wifi_csi_config_t ──
    wifi_csi_config_t csi_cfg = {
        .lltf_en           = true,   // L-LTF CSI
        .htltf_en          = true,   // HT-LTF CSI
        .stbc_htltf2_en    = true,   // STBC HT-LTF2
        .ltf_merge_en      = true,   // 合并 LTF
        .channel_filter_en = false,  // 不过滤信道
        .manu_scale        = false,
    };
#endif

    ESP_ERROR_CHECK(esp_wifi_set_csi_config(&csi_cfg));
    ESP_ERROR_CHECK(esp_wifi_set_csi_rx_cb(wifi_csi_cb, NULL));
    ESP_ERROR_CHECK(esp_wifi_set_csi(true));
}
```

**解析：** ESP32-C6（WiFi 6 芯片）使用全新的 `wifi_csi_acquire_config_t` 结构体，字段更多（支持 HE/SU/MU/DCM/Beamforming 等 WiFi 6 特性）。原版 ESP32 使用 `wifi_csi_config_t`，字段名不同（`lltf_en`/`htltf_en`）。`acquire_csi_ht40 = 0` 禁用 HT40 是有意为之——HT40 会把 CSI 缓冲区翻倍，导致子载波数不可预测，禁用后 `n_sub` 稳定，算法逻辑更可靠。`esp_wifi_set_csi_rx_cb` 注册回调，`esp_wifi_set_csi(true)` 启动 CSI 采集。

### 4.6 Flipper 端 UART 协议解析—— `csight_uart.c`

Flipper 侧用一个独立线程持续接收并解析 ESP32 发来的协议帧：

```c
static int32_t uart_rx_thread(void* ctx) {
    CSIghtApp* app = (CSIghtApp*)ctx;
    uint8_t    buf[CSIGHT_RX_BUF];

    while(true) {
        // 检查停止标志
        uint32_t flags = furi_thread_flags_get();
        if(flags & THREAD_STOP_FLAG) {
            furi_thread_flags_clear(THREAD_STOP_FLAG);
            break;
        }

        // 从流缓冲区读取（50ms 超时以便定期检查停止标志）
        size_t len = furi_stream_buffer_receive(
            app->rx_stream, buf, sizeof(buf), furi_ms_to_ticks(50));
        if(len == 0) continue;

        size_t i = 0;
        while(i < len) {
            uint8_t hdr = buf[i++];

            switch(hdr) {
                case PROTO_HANDSHAKE_ACK: {
                    // [0xBB][payload_len][chip_name\0][csi_support][fw_maj][fw_min][chk]
                    if(i >= len) break;
                    uint8_t payload_len = buf[i++];
                    if(i + payload_len > len) break;

                    // 手动 strnlen（Flipper SDK 无此函数）
                    size_t name_len = 0;
                    while(name_len < payload_len && buf[i + name_len] != '\0') name_len++;
                    if(name_len >= sizeof(app->chip_name)) name_len = sizeof(app->chip_name) - 1;

                    memcpy(app->chip_name, &buf[i], name_len);
                    app->chip_name[name_len] = '\0';
                    i += name_len + 1;

                    if(i < len) app->csi_support = buf[i++];
                    if(i < len) app->fw_major    = buf[i++];
                    if(i < len) app->fw_minor     = buf[i++];
                    if(i < len) i++; // 跳过校验

                    app->state = AppStateCompatCheck;  // 进入兼容性检查
                    break;
                }

                case PROTO_MOTION_EVT: {
                    // [0x01][intensity][proximity][chk]
                    if(i + 2 > len) break;
                    uint8_t intensity = buf[i++];
                    uint8_t proximity = buf[i++];
                    if(i < len) i++; // 跳过校验

                    app->motion_intensity = intensity;
                    app->proximity        = proximity;
                    csight_add_blip(app, intensity, proximity);
                    app->target_acquired = true;
                    app->target_ts       = furi_get_tick();
                    break;
                }

                case PROTO_WATERFALL_EVT: {
                    // [0x03][count][bin0..binN][chk]
                    if(i >= len) break;
                    uint8_t count = buf[i++];
                    if(i + count > len) break;

                    // 把 count 个 bin 压缩为 WATERFALL_HEIGHT/4 个像素
                    uint8_t col[WATERFALL_HEIGHT / 4];
                    memset(col, 0, sizeof(col));
                    int bins_per_pixel = count / (WATERFALL_HEIGHT / 4);
                    if(bins_per_pixel < 1) bins_per_pixel = 1;

                    for(int p = 0; p < WATERFALL_HEIGHT / 4; p++) {
                        uint32_t sum = 0;
                        for(int b = 0; b < bins_per_pixel; b++) {
                            int idx = p * bins_per_pixel + b;
                            if(idx < (int)count) sum += buf[i + idx];
                        }
                        col[p] = (uint8_t)(sum / (uint32_t)bins_per_pixel);
                    }

                    memcpy(app->waterfall.cols[app->wf_write_col], col, sizeof(col));
                    app->wf_write_col = (app->wf_write_col + 1) % WATERFALL_COLS;  // 环形缓冲
                    i += count;
                    if(i < len) i++;
                    break;
                }
            }
        }
    }
    return 0;
}
```

**逐段解析：**

- **流式协议解析**：Flipper 逐字节读取帧头，根据帧头类型解析后续字段。每个分支都有 `if(i >= len) break` 边界检查——防止跨帧数据不完整时越界读取。这是嵌入式串口协议解析的健壮性范式。
- **手动 strnlen**：Flipper SDK 不提供 `strnlen`，作者手写了一个有限长度的字符串搜索循环，并用 `sizeof(app->chip_name) - 1` 截断防止溢出。
- **瀑布环形缓冲**：`wf_write_col = (wf_write_col + 1) % WATERFALL_COLS` 实现环形缓冲区，新数据覆盖最旧列，实现瀑布图的"滚动"效果。
- **线程安全退出**：50ms 超时的 `furi_stream_buffer_receive` 保证线程不会永久阻塞，能定期检查 `THREAD_STOP_FLAG` 实现优雅退出。

### 4.7 雷达光点定位算法—— `csight_app.c`

收到运动事件后，Flipper 把运动强度和距离转化为雷达屏幕上的光点坐标：

```c
void csight_add_blip(CSIghtApp* app, uint8_t intensity, uint8_t proximity) {
    // 找到最旧的光点槽位（age 最小）来复用
    int oldest = 0;
    for(int i = 1; i < MAX_BLIPS; i++) {
        if(app->blips[i].age < app->blips[oldest].age) oldest = i;
    }

    uint8_t a  = app->sweep_angle & 63;           // 当前扫描角度 (0-63)
    uint8_t ja = (a + (intensity & 3)) & 63;       // 加微小抖动避免光点重叠
    // proximity 越高(目标越近) → dist 越小(光点越靠近雷达中心)
    int dist   = 2 + ((100 - proximity) * (RADAR_R - 4)) / 100;

    // 用查找表计算坐标（避免浮点运算）
    app->blips[oldest].x         = (int8_t)((COS64[ja] * dist) / 59);
    app->blips[oldest].y         = (int8_t)((SIN64[ja] * dist) / 59);
    app->blips[oldest].age       = 255;            // 重置存活时间
    app->blips[oldest].intensity = intensity;
}
```

**逐段解析：**

- **光点复用**：`MAX_BLIPS = 8`，只保留 8 个光点。新光点覆盖最旧（`age` 最小）的槽位，形成"旧光点淡出、新光点出现"的雷达效果。
- **距离映射**：`dist = 2 + ((100 - proximity) × 24) / 100`。`proximity=100`（最近）时 `dist=2`（中心附近）；`proximity=0`（最远）时 `dist=26`（雷达边缘）。这让近的目标显示在雷达中心、远的显示在外圈，符合直觉。
- **三角函数查找表**：`COS64`/`SIN64` 是 64 步（每步 5.625°）的 `int8_t` 查找表，值域经 `/59` 缩放。这避免了 Flipper 的浮点运算——用整数乘除查表即可计算极坐标到直角坐标的转换，在资源受限的 MCU 上极为高效。
- **角度抖动**：`ja = (a + (intensity & 3)) & 63` 把运动强度的低 2 位作为角度微抖动，让多个同时检测到的运动光点不完全重叠。

### 4.8 配置持久化—— `csight_app.c`

用户选择的板卡预设、引脚、灵敏度等配置存入 Flipper SD 卡，下次启动自动加载：

```c
#define CONFIG_MAGIC 0xC5   // 配置有效性魔数

typedef struct {
    uint8_t magic;        // 0xC5 = 配置有效（首次运行检测）
    uint8_t preset_idx;   // 板卡预设索引
    uint8_t tx_pin;       // ESP32 TX 引脚
    uint8_t rx_pin;       // ESP32 RX 引脚
    uint8_t sensitivity;  // 灵敏度 0-10
    uint8_t display_mode; // 显示模式
    uint8_t _pad[2];
} CSIghtConfig;

void csight_config_save(CSIghtApp* app) {
    Storage* storage = furi_record_open(RECORD_STORAGE);
    storage_simply_mkdir(storage, EXT_PATH("apps_data/csight"));

    File* file = storage_file_alloc(storage);
    if(storage_file_open(file, CONFIG_PATH, FSAM_WRITE, FSOM_CREATE_ALWAYS)) {
        CSIghtConfig cfg = {
            .magic        = CONFIG_MAGIC,
            .preset_idx   = app->preset_idx,
            .tx_pin       = app->tx_pin,
            .rx_pin       = app->rx_pin,
            .sensitivity  = app->sensitivity,
            .display_mode = (uint8_t)app->display_mode,
        };
        storage_file_write(file, &cfg, sizeof(cfg));
    }
    storage_file_close(file);
    storage_file_free(file);
    furi_record_close(RECORD_STORAGE);
}

bool csight_config_load(CSIghtApp* app) {
    // ... 打开文件读取 ...
    if(storage_file_read(file, &cfg, sizeof(cfg)) == sizeof(cfg)
       && cfg.magic == CONFIG_MAGIC) {
        // 校验魔数 + 范围检查后加载
        app->preset_idx   = cfg.preset_idx < BOARD_PRESET_COUNT ? cfg.preset_idx : 0;
        app->sensitivity  = cfg.sensitivity <= 10 ? cfg.sensitivity : 5;
        // ...
        loaded = true;
    }
    if(!loaded) {
        // 首次运行使用默认值
        app->sensitivity  = 5;
        app->display_mode = DisplayModeRadar;
    }
}
```

**解析：** `CONFIG_MAGIC = 0xC5` 用于区分"首次运行（无配置文件）"与"已有配置"。加载时对每个字段做范围检查（`preset_idx < BOARD_PRESET_COUNT`、`sensitivity <= 10`），防止损坏的配置文件导致越界。`_pad[2]` 保证结构体 8 字节对齐。这是嵌入式配置持久化的标准范式。

---

## 五、API 使用指南

### UART 协议完整定义

| 方向 | 命令头 | 帧格式 | 说明 |
|---|---|---|---|
| Flipper→ESP32 | `0xAA` | `[0xAA]` | 握手请求 |
| ESP32→Flipper | `0xBB` | `[0xBB][len][name\0][csi][fw_maj][fw_min][xor]` | 握手应答 |
| ESP32→Flipper | `0x01` | `[0x01][intensity][proximity][xor]` | 运动事件 |
| ESP32→Flipper | `0x03` | `[0x03][count][bins...][xor]` | 瀑布数据 |
| Flipper→ESP32 | `0x10` | `[0x10][sensitivity]` | 设置灵敏度（0-10） |
| Flipper→ESP32 | `0x11` | `[0x11][mode]` | 设置模式（0=运动,1=瀑布,2=距离） |

### Python 串口监听示例

可以用 Python 直接连接 ESP32 监听运动事件（无需 Flipper）：

```python
import serial, struct

ser = serial.Serial("/dev/ttyUSB0", 115200, timeout=1)

# 发送握手请求
ser.write(bytes([0xAA]))

# 读取握手应答
ack = ser.read(64)
if ack and ack[0] == 0xBB:
    name_end = ack.index(0, 2)
    chip_name = ack[2:name_end].decode()
    csi_support = ack[name_end + 1]
    print(f"芯片: {chip_name}, CSI 支持: {csi_support}")

# 设置灵敏度 7、模式 0（运动检测）
ser.write(bytes([0x10, 7]))
ser.write(bytes([0x11, 0]))

# 监听运动事件
print("等待运动...")
while True:
    data = ser.read(4)
    if len(data) == 4 and data[0] == 0x01:
        intensity = data[1]
        proximity = data[2]
        chk = data[0] ^ data[1] ^ data[2]
        if chk == data[3]:
            print(f"检测到运动! 强度:{intensity} 距离:{proximity}%")
```

### cURL 触发 Flipper 端 Web UI（v1.4 路线图）

未来版本计划支持独立 Web UI，可通过 HTTP 控制：

```bash
# 切换显示模式
curl -X POST http://flipper.local/api/mode -d '{"mode":"waterfall"}'

# 调整灵敏度
curl -X POST http://flipper.local/api/sensitivity -d '{"level":8}'
```

---

## 六、编译与部署

### 环境搭建

| 项目 | 要求 |
|---|---|
| ESP32 固件 | ESP-IDF v5.2+ |
| Flipper FAP | ufbt（Flipper 构建工具） |
| 支持芯片 | ESP32 / S3 / C3 / C6（推荐 S3 或 C6） |
| Flipper 固件 | Official / Momentum / Unleashed |

### 编译配置

`sdkconfig.defaults` 关键项：

| 配置项 | 说明 |
|---|---|
| `CONFIG_IDF_TARGET` | 目标芯片（esp32/esp32s3/esp32c3/esp32c6） |
| WiFi CSI | 通过 `esp_wifi_set_csi` 启用 |

### 烧录步骤

**ESP32 固件烧录（esptool 命令行）：**

```bash
pip install esptool

esptool.py --chip auto --port /dev/ttyUSB0 --baud 460800 \
  write_flash \
  0x1000 bootloader.bin \
  0x8000 partition-table.bin \
  0x10000 csight_esp32.bin
```

| 文件 | 烧录地址 |
|---|---|
| `bootloader.bin` | `0x1000` |
| `partition-table.bin` | `0x8000` |
| `csight_esp32.bin` | `0x10000` |

**Flipper FAP 安装：**

```bash
# 从源码编译
cd flipper
ufbt

# 或直接下载预编译 FAP，拷贝到 SD 卡
cp csight.fap /SD_CARD/apps/GPIO/csight.fap
# 从 Apps → GPIO → CSIght 启动
```

**接线与使用：**

1. 给 ESP32 板卡供电（3.3V）
2. 按 ESP32 TX→Flipper Pin14、RX→Flipper Pin13、GND→GND 接线
3. 在 Flipper 上启动 CSIght，确认 ESP32 已上电后按 OK
4. 等待握手完成，选择板卡预设（或自动加载已存配置）
5. 进入扫描模式，用 ◄► 切换显示模式，▲▼ 调整灵敏度

---

## 七、项目亮点与适用场景

### 项目亮点

1. **WiFi CSI 感知平民化**：把原本需要专用设备（如 Intel 5300 NIC + Atheros CSI Tool）的 WiFi 感知技术，用几美元的 ESP32 实现，极大降低了技术门槛。
2. **算法设计精巧**：基线校准 + EMA 自适应滤波的组合，既保证了对突发运动的灵敏检测，又抵御了环境慢漂移导致的误报，工程上非常实用。
3. **双设备解耦架构**：ESP32 做感知、Flipper 做交互，通过精简 UART 协议连接，职责清晰、各自独立升级。
4. **广泛硬件兼容**：50+ 板卡预设 + 自动芯片检测 + 自定义引脚，几乎覆盖所有常见 ESP32 开发板。
5. **整数优化渲染**：雷达光点定位用 64 步三角函数查找表 + 整数运算，在 Flipper Zero 有限资源下流畅运行。

### 适用场景

| 场景 | 说明 |
|---|---|
| 安防感知 | 无摄像头隐私问题的运动检测，适合卧室/卫生间等隐私区域 |
| 智能家居 | 人体存在检测，联动灯光/空调（无隐私担忧） |
| 老人看护 | 跌倒/异常静止检测，穿墙感知无需在房间内安装设备 |
| 嵌入式教学 | 学习 WiFi CSI 原理、信号处理算法、UART 协议设计 |
| 安全研究 | 研究 RF 感知的隐私边界与防御方法 |
| 原型开发 | 快速验证 WiFi 感知概念，无需昂贵专用硬件 |

---

## 八、总结

CSIght 是一个将前沿学术概念（WiFi CSI 感知）落地为消费级可玩产品的优秀范例。它没有停留在论文层面，而是用 ESP32 + Flipper Zero 这两个爱好者手边常见的设备，实现了真正可用的穿墙运动检测。

从代码质量看，CSI 核心算法（基线校准 + EMA 自适应 + 阈值检测）虽然简洁但原理扎实，UCLA 等机构的 WiFi 感知论文中也是类似思路。协议设计紧凑健壮，多芯片适配考虑周全，配置持久化规范。Flipper 端的整数三角函数渲染和状态机设计也体现了嵌入式优化的功底。

项目的路线图令人期待——呼吸检测、多节点定向雷达、ESP32-C5 支持等，如果实现将进一步拓展 WiFi 感知的能力边界。对于想深入理解"WiFi 信号如何感知环境"的开发者，CSIght 是一个绝佳的入门与实践样本。

---

📝 作者：蔡浩宇（jun-chy）  
📅 日期：2026-07-09  
🔗 项目地址：https://github.com/joelewis012/CSIght
