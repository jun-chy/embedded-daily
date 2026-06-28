# DSPi：把树莓派 Pico 变成专业级音频 DSP 处理器——深度源码解析

> **日期**：2026-06-28  
> **分类**：RP2040 / RP2350 · 数字音频处理  
> **Star 数**：1011 ⭐  
> **作者**：蔡浩宇（jun-chy）

---

## 一、项目链接

| 属性 | 信息 |
|---|---|
| **GitHub 地址** | https://github.com/WeebLabs/DSPi |
| **作者** | WeebLabs |
| **许可证** | GPL-3.0 |
| **最近更新** | 2026-06-28 |
| **创建时间** | 2026-01-08 |
| **主要语言** | C（含手写 ARM 汇编） |
| **Fork 数** | 49 |
| **仓库大小** | 3.6 MB |

---

## 二、项目简介

DSPi 是一个把 Raspberry Pi Pico（RP2040）或 Pico 2（RP2350）变成专业级数字音频处理器的固件项目。它让这块不到一杯咖啡价格的微控制器板子充当一块 **USB 声卡 + 板载 DSP 引擎**，提供房间校正、有源分频、参数均衡（PEQ）、时间对齐、响度补偿和耳机串扰消除等核心能力。

它的设计目标是让 RP2040/RP2350 成为"音频界的瑞士军刀"。固件实现了完整的 USB Audio Class 1 设备栈，支持 16/24-bit PCM、44.1/48/96 kHz 采样率，并通过 PIO（可编程 IO）硬件同时输出 S/PDIF、I2S 和 PDM 多路音频。整套 DSP 流水线在双核上并行运行：RP2040 采用 Q28 定点运算配合手写 ARM 汇编优化，RP2350 则利用硬件浮点单元采用混合 SVF/biquad 架构。

应用场景涵盖：HiFi 有源音箱分频、耳机放大器均衡、家庭影院低音管理、录音棚监听校正、便携音频处理器等。

---

## 三、核心特性一览表

| 特性 | RP2040 (Pico) | RP2350 (Pico 2) |
|---|---|---|
| **系统时钟** | 307.2 MHz（超频） | 307.2 MHz |
| **音频运算** | Q28 定点 | 单精度浮点 |
| **EQ 频段** | 每通道 10 段（共 70 段） | 每通道 10 段（共 110 段） |
| **总通道数** | 7（2 主 + 4 S/PDIF/I2S + 1 PDM） | 11（2 主 + 8 S/PDIF/I2S + 1 PDM） |
| **输出插槽** | 2 立体声 | 4 立体声 |
| **输出位深** | 24-bit | 24-bit |
| **PDM 输出** | 1（超低音） | 1（超低音） |
| **最大延迟** | 每输出 21ms | 每输出 42ms |
| **数学引擎** | 手写 ARM 汇编 | 硬件 FPU（混合 SVF/biquad） |
| **双核 EQ** | Core 1 处理输出 3-4 | Core 1 处理输出 3-8 |
| **用户预设** | 10 个槽位 | 10 个槽位 |
| **USB 即插即用** | macOS/Windows/Linux/iOS | 同左 |

---

## 四、硬件架构

### 4.1 系统组成图

```
                    ┌─────────────────────────────────┐
                    │          USB Host (PC/Mac)        │
                    │   16/24-bit PCM 44.1/48/96 kHz    │
                    └───────────────┬───────────────────┘
                                    │ USB (TinyUSB UAC1)
                    ┌───────────────▼───────────────────┐
                    │        RP2040 / RP2350             │
                    │  ┌─────────┐    ┌──────────────┐   │
                    │  │ Core 0  │    │   Core 1     │   │
                    │  │ 主流水线 │◄──►│ EQ Worker /  │   │
                    │  │ USB RX  │    │ PDM 调制器    │   │
                    │  └────┬────┘    └──────┬───────┘   │
                    │       │  DSP Pipeline  │           │
                    │  Preamp→EQ→Crossfeed   │           │
                    │  →Loudness→Mixer→Delay │           │
                    │       │                │           │
                    │  ┌────▼────────────────▼──────┐    │
                    │  │   PIO0          PIO1       │    │
                    │  │ S/PDIF×2/I2S   PDM Sub     │    │
                    │  └────┬──────────────┬────────┘    │
                    └───────┼──────────────┼─────────────┘
                            │              │
                   GPIO 6/7 │     GPIO 10  │
                   GPIO14/15(I2S BCK/LRCLK)│
                            │              │
              ┌─────────────▼──┐   ┌───────▼────────┐
              │  DAC / 功放     │   │  RC 低通滤波    │
              │  (I2S 或 S/PDIF)│   │  → 有源低音炮   │
              └────────────────┘   └────────────────┘
```

### 4.2 引脚配置（RP2040）

| 功能 | 引脚 | 说明 |
|---|---|---|
| S/PDIF / I2S 输出槽 0（Out 1-2） | GPIO 6 | 主左右声道或分频对 1 |
| S/PDIF / I2S 输出槽 1（Out 3-4） | GPIO 7 | 分频对 2 |
| PDM 超低音输出（Out 5） | GPIO 10 | 有源低音炮，需 RC 低通 |
| I2S BCK（共享） | GPIO 14 | 位时钟 |
| I2S LRCLK | GPIO 15 | 帧时钟（固定为 BCK+1） |
| I2S MCK（可选） | GPIO 21 | 128×/256× Fs 主时钟 |
| USB | Micro-USB | 连接主机 |

### 4.3 硬件清单

| 组件 | 型号 | 用途 |
|---|---|---|
| 主控板 | Raspberry Pi Pico (RP2040) 或 Pico 2 (RP2350) | 核心处理 |
| DAC（可选） | 任意 I2S DAC（如 PCM5102） | 数字转模拟 |
| 光纤发射器 | Toshiba TX179（可选） | S/PDIF 光纤输出 |
| 低通滤波 | 电阻 + 电容 | PDM 转模拟（低音炮） |
| USB 线 | Micro-USB | 连接主机 |

---

## 五、固件架构

### 5.1 文件结构

| 文件 | 职责 |
|---|---|
| `main.c` | 主入口、USB 任务循环、采样率切换、闪存操作 |
| `dsp_pipeline.c` | DSP 流水线核心：biquad/SVF 系数计算、滤波处理 |
| `dsp_process_rp2040.S` | RP2040 专用定点 EQ 汇编优化 |
| `audio_pipeline.c` | 音频流水线编排：Preamp→EQ→Crossfeed→Mixer→Delay |
| `usb_audio.c` | USB Audio Class 1 类驱动、反馈控制 |
| `crossfeed.c` | BS2B 耳机串扰消除 |
| `loudness.c` | ISO 226 响度补偿 |
| `leveller.c` | RMS 音量均衡器（上行压缩） |
| `pdm_generator.c` | PDM 超低音输出、二阶 Sigma-Delta 调制 |
| `config.h` | 全局配置：引脚、缓冲区、通道数、剪辑阈值 |
| `flash_storage.c` | 预设持久化（闪存读写） |
| `vendor_commands.c` | USB Vendor 命令处理（WinUSB 控制协议） |

### 5.2 模块职责说明

固件采用**双核分工**架构。Core 0 运行主循环：处理 USB 枚举、接收 USB 音频包、执行主流水线（Preamp → Master EQ → Crossfeed → Loudness → Matrix Mixer → Output EQ → Gain/Delay），并驱动 S/PDIF/I2S 的 PIO 输出。Core 1 根据模式切换：在 PDM 模式下运行二阶 Sigma-Delta 调制器生成超低音 PDM 码流；在 EQ Worker 模式下并行处理分配给它的输出通道的 EQ + 增益 + 延迟，与 Core 0 形成流水线并行。

---

## 六、核心代码深度分析

### 6.1 系统初始化流程（main.c）

DSPi 的 `main()` 函数是整个系统的入口，负责硬件初始化、时钟配置、双核启动和主循环调度：

```c
int main(void) {
    // Initial LED on to show we're alive
    gpio_init(25); gpio_set_dir(25, GPIO_OUT);
    gpio_put(25, 1);

#if !PICO_RP2350
    set_sys_clock_pll(1536000000, 4, 2);
#endif

    core0_init();

    // Enable watchdog
    watchdog_enable(8000, 1);

    while (1) {
        // Update watchdog
        watchdog_update();

        // TinyUSB device task — processes enumeration, control transfers, and
        // deferred bus events. Must be called at least once per main-loop iteration.
        tud_task();

        // Fire any queued device→host notifications to EP 0x83.
        usb_notify_tick();

        // LG Sound Sync detection tick
        lg_sound_sync_tick();

        // DAC hardware-mute deadline check
        dac_hw_mute_tick();

        // Drain USB audio ring — highest priority (only when USB is active input).
        if (active_input_source == INPUT_SOURCE_USB) {
            usb_audio_drain_ring();
        } else {
            usb_audio_flush_ring();
        }

        // Poll SPDIF input when active
        if (active_input_source == INPUT_SOURCE_SPDIF) {
            SpdifInputState rx_state = spdif_input_get_state();
            // ... SPDIF lock acquisition / prefill / lock-loss handling ...
            spdif_input_poll();
            spdif_input_update_clock_servo();
        }

        // Handle deferred flash SET commands (fire-and-forget, no result).
        // ... preset name / startup slot / output config writes ...
    }
}
```

**逐段解析**：

- `gpio_init(25)` 点亮板载 LED 作为"活着"的心跳指示，这是裸机固件最朴素的调试手段。
- `set_sys_clock_pll(1536000000, 4, 2)` 是 RP2040 的**超频核心**：VCO 设为 1536 MHz，除以 4 再除以 2 得到 307.2 MHz。这个频率不是随便选的——307.2 MHz 能让 48 kHz 采样率下所有 PIO 分频器都是整数（307.2M / 48k = 6400），避免小数分频带来的时钟抖动。RP2350 因架构不同省略了此调用。
- `core0_init()` 完成全部子系统初始化：PIO 程序加载、DMA 通道申请、USB 描述符注册、闪存预设加载、DSP 滤波器初始化，并通过 `multicore_launch_core1(pdm_core1_entry)` 启动 Core 1。
- `watchdog_enable(8000, 1)` 开启 8 秒看门狗，`1` 表示暂停时也运行。主循环每次迭代 `watchdog_update()` 喂狗，确保固件死锁时能自动复位。
- 主循环中 `tud_task()` 处理 TinyUSB 设备枚举和控制传输；音频数据接收实际由 USB 中断中的 UAC1 回调完成，因此 `tud_task()` 对音频流本身不是延迟敏感的。
- `usb_audio_drain_ring()` 是音频处理的关键路径：USB ISR 将原始包推入环形缓冲区，主循环在这里排空并运行完整 DSP 流水线，避免在 USB IRQ 上下文中做重计算。
- SPDIF 输入路径包含精巧的**锁获取状态机**：检测到锁定后先排空输出、预填充消费缓冲到 50%，再同步启动所有输出，最后释放 DAC 硬件静音——这套握手流程确保从 USB 切换到 SPDIF 输入时无缝、无爆音。

### 6.2 采样率切换（perform_rate_change）

当主机请求新的采样率时，固件必须原子地切换所有相关参数。这是音频系统中最容易出爆音的地方：

```c
static void perform_rate_change(uint32_t new_freq) {
    switch (new_freq) { case 44100: case 48000: case 96000: break; default: new_freq = 44100; }

    // Engage mute and wait for Core 1 EQ worker to drain before touching
    // filter coefficients or PIO dividers.
    prepare_pipeline_reset(PRESET_MUTE_SAMPLES);

    // Update the audio format so pico_audio_spdif can update the PIO divider
    audio_format_48k.sample_freq = new_freq;

    // Reset sync
    extern volatile bool sync_started;
    extern volatile uint64_t total_samples_produced;
    sync_started = false;
    total_samples_produced = 0;

    // Pre-compute nominal feedback and reset controller
    nominal_feedback_10_14 = ((uint64_t)new_freq << 14) / 1000;
    feedback_10_14 = nominal_feedback_10_14;
    reset_usb_feedback_loop();

    dsp_recalculate_all_filters((float)new_freq);
    loudness_recompute_pending = true;
    crossfeed_update_pending = true;
    leveller_update_pending = true;
    pdm_update_clock(new_freq);

    // Atomically update all I2S instances and restart in sync
    audio_i2s_update_all_frequencies(new_freq);

    if (i2s_mck_enabled) {
        audio_i2s_mck_update_frequency(new_freq, i2s_mck_multiplier);
    }

    if (!spdif_prefilling) {
        complete_pipeline_reset();
    }
}
```

**逐段解析**：

- 第一行做**输入验证**：只允许 44.1/48/96 kHz 三种采样率，其余强制回退到 44.1 kHz，防止恶意或错误请求导致 PIO 分频器出现非整数。
- `prepare_pipeline_reset(PRESET_MUTE_SAMPLES)` 是关键：先静音若干样本并等待 Core 1 EQ Worker 排空，再动滤波器系数或 PIO 分频器。注释解释得很清楚——不这样做，旧采样率的消费缓冲会在新位时钟下播放约 16ms，产生可听见的音高偏移和重同步爆音。
- `nominal_feedback_10_14 = ((uint64_t)new_freq << 14) / 1000` 计算 USB 同步反馈值。这里用 10.14 定点格式（USB Audio Class 标准），将采样率左移 14 位再除以 1000（因为 USB 反馈以每毫秒样本数为单位）。
- `dsp_recalculate_all_filters()` 用新采样率重算所有 biquad/SVF 系数；同时标记 `loudness_recompute_pending`、`crossfeed_update_pending`、`leveller_update_pending` 让各子系统在下一个安全点重算系数。
- `audio_i2s_update_all_frequencies()` 原子地更新所有 I2S 实例的 PIO 分频器并同步重启，避免主从分频器短暂不匹配。
- `complete_pipeline_reset()` 排空所有消费池（旧采样率音频）并以新分频器同步重启输出——除非 SPDIF 锁获取预填充正在进行中。

### 6.3 DSP 滤波器系数计算（dsp_pipeline.c）

这是整个项目的数学核心。`dsp_compute_coefficients()` 根据滤波器类型、频率、Q 值和增益，计算 biquad 或 SVF 系数：

```c
void dsp_compute_coefficients(EqParamPacket *p, Biquad *bq, float sample_rate) {
    bool user_bypass = (p->bypass == 1);

    if (user_bypass || is_filter_flat(p) || sample_rate == 0) {
        bq->bypass = true;
#if PICO_RP2350
        bq->b0 = 1.0f; bq->b1 = 0.0f; bq->b2 = 0.0f; bq->a1 = 0.0f; bq->a2 = 0.0f;
        bq->use_svf = false;
#else
        bq->b0 = 1 << FILTER_SHIFT; bq->b1 = 0; bq->b2 = 0; bq->a1 = 0; bq->a2 = 0;
#endif
        return;
    }

    bq->bypass = false;

    // Input validation
    if (p->Q < 0.1f) p->Q = 0.1f;
    if (p->Q > 20.0f) p->Q = 20.0f;
    if (p->freq < 10.0f) p->freq = 10.0f;
    if (p->freq > sample_rate * 0.45f) p->freq = sample_rate * 0.45f;

    float A = powf(10.0f, p->gain_db / 40.0f);
```

**逐段解析**：

- `is_filter_flat()` 是一个优化函数：判断滤波器是否实际为平坦（无效果）。对于增益接近 0 dB（`fabsf(gain_db) < 0.01f`）的 peaking/shelf 滤波器，直接标记 bypass，跳过后续所有计算——这避免了在 EQ 频段未启用时浪费 CPU 周期。
- bypass 路径中，RP2040 把 `b0` 设为 `1 << FILTER_SHIFT`（即 Q28 格式的 1.0），RP2350 设为 `1.0f`。这种条件编译让同一份代码在两个平台上都正确工作。
- 输入验证钳制 Q 在 0.1~20、频率在 10 Hz ~ 0.45×Fs 之间，防止数值不稳定。`0.45` 而非 `0.5` 留了 10% 的奈奎斯特余量。
- `A = powf(10.0f, gain_db / 40.0f)` 计算增益的线性幅度比。注意除以 40 而非 20——因为 RBJ Audio EQ Cookbook 公式中 A 是幅度比的平方根，所以 `10^(dB/40)` 而非 `10^(dB/20)`。

接下来是 RP2350 独有的 **SVF/biquad 混合架构**决策：

```c
#if PICO_RP2350
    // SVF/biquad crossover decision + state reset on path change
    bool was_svf = bq->use_svf;
    bq->use_svf = (p->freq < (sample_rate / 7.5f));
    if (was_svf != bq->use_svf) {
        bq->s1 = 0.0f; bq->s2 = 0.0f;
        bq->svic1eq = 0.0f; bq->svic2eq = 0.0f;
    }

    if (bq->use_svf) {
        // SVF coefficients (Simper, "SvfLinearTrapAllOutputs", Cytomic 2021)
        float g = tanf(3.1415926535f * p->freq / sample_rate);
        float k = 1.0f / p->Q;
        // ... per-type k and g adjustments ...
        float sva1_f = 1.0f / (1.0f + g * (g + k));
        float sva2_f = g * sva1_f;
        float sva3_f = g * sva2_f;
        // ... per-type mixer coefficients m0, m1, m2 ...
        return;
    }
#endif
```

**逐段解析**：

- `bq->use_svf = (p->freq < (sample_rate / 7.5f))` 是**核心架构决策**：当截止频率低于采样率的 1/7.5（48 kHz 下约 6.4 kHz）时使用 SVF（State Variable Filter）拓扑，否则用传统 biquad。原因是 biquad 在低频/高采样率比下会出现系数精度问题（b1、b2 接近 a1、a2 导致相消），而 SVF 在低频具有数值优势。这是音频 DSP 的经典权衡。
- 切换 SVF/biquad 路径时必须清零滤波器状态（`s1, s2, svic1eq, svic2eq`），否则旧拓扑的积分器残留会作为瞬态注入新拓扑。
- SVF 系数采用 Andrew Simper（Cytomic）2021 年发表的线性陷阱全输出公式，这是当前业界公认的数值稳定性最好的 SVF 实现。`g = tan(π·f/Fs)` 是预扭曲频率，`k = 1/Q` 是阻尼。

biquad 系数计算则遵循经典的 RBJ Audio EQ Cookbook：

```c
    float omega = 2.0f * 3.1415926535f * p->freq / sample_rate;
    float sn = sinf(omega); float cs = cosf(omega);
    float alpha = sn / (2.0f * p->Q);
    // ...
    switch (p->type) {
        case FILTER_LOWPASS: b0_f = (1-cs)/2; b1_f = 1-cs; b2_f = (1-cs)/2;
                             a0_f = 1+alpha; a1_f = -2*cs; a2_f = 1-alpha; break;
        case FILTER_PEAKING: b0_f = 1+alpha*A; b1_f = -2*cs; b2_f = 1-alpha*A;
                             a0_f = 1+alpha/A; a1_f = -2*cs; a2_f = 1-alpha/A; break;
        // ...
    }

#if PICO_RP2350
    float inv_a0 = 1.0f / a0_f;
    bq->b0 = b0_f * inv_a0;
    // ... float storage ...
#else
    // Q28 Fixed Point Storage
    float scale = (float)(1LL << FILTER_SHIFT);
    bq->b0 = (int32_t)((b0_f / a0_f) * scale);
    // ... Q28 storage ...
#endif
```

**逐段解析**：

- `alpha = sn / (2.0f * p->Q)` 是 RBJ 公式中的带宽参数，Q 越大 alpha 越小（滤波器越窄）。
- 系数归一化（除以 `a0`）后，RP2350 直接存浮点，RP2040 转为 Q28 定点（乘以 `2^28` 后取整）。`FILTER_SHIFT = 28` 意味着 4 位整数 + 28 位小数，动态范围 ±8.0 足够覆盖音频滤波器系数（通常在 ±2 以内）。

### 6.4 定点快速乘法（RP2040 专用）

RP2040 没有硬件除法器和单周期 32×32 乘法，Q28 定点乘法需要特殊处理：

```c
DSP_TIME_CRITICAL int32_t fast_mul_q28(int32_t a, int32_t b) {
    int32_t ah = a >> 16;
    uint32_t al = a & 0xFFFF;
    int32_t bh = b >> 16;
    uint32_t bl = b & 0xFFFF;

    int32_t high = ah * bh;
    int32_t mid1 = ah * bl;
    int32_t mid2 = al * bh;

    return (high << 4) + ((mid1 + mid2) >> 12);
}
```

**逐段解析**：

- 这是经典的 **16×16 分块乘法**优化。两个 Q28 数相乘，结果应为 Q56，需要右移 28 位回 Q28。
- 将 32 位数拆成高 16 位和低 16 位，做三次 16×16 乘法（RP2040 的 MUL 指令是单周期的）：`ah*bh`（高×高）、`ah*bl`（高×低）、`al*bh`（低×高）。最低 16×16 项 `al*bl` 在 Q28 精度下可忽略。
- `high << 4`：高×高项原本在 Q56 的最高 32 位，左移 4 位对齐到 Q28 的位置（因为高 16 位已含 28 位中的 16 位整数部分偏移）。
- `(mid1 + mid2) >> 12`：两个交叉项在 Q44 位置，右移 12 位对齐到 Q32，再与 high 项相加完成 Q28 结果。
- `DSP_TIME_CRITICAL` 宏将函数放入 RAM（而非 Flash XIP），避免 Flash 访问延迟影响实时性能。实际上 RP2040 还有手写汇编版本（`dsp_process_rp2040.S`）进一步压榨性能。

### 6.5 SVF 滤波处理（RP2350 浮点路径）

滤波处理函数对每个样本依次通过所有活跃频段。RP2350 的块处理版本针对不同滤波器类型做了特化优化：

```c
DSP_TIME_CRITICAL
void dsp_process_channel_block(Biquad * __restrict biquads, float * __restrict samples,
                               uint32_t count, uint8_t channel) {
    uint8_t num_bands = channel_band_counts[channel];

    for (int band = 0; band < num_bands; band++) {
        Biquad *bq = &biquads[band];
        if (bq->bypass) continue;

        if (bq->use_svf) {
            float a1 = bq->sva1, a2 = bq->sva2, a3 = bq->sva3;
            float m0 = bq->svm0, m1 = bq->svm1, m2 = bq->svm2;
            float ic1eq = bq->svic1eq, ic2eq = bq->svic2eq;
            float *sp = samples;

            // Per-type specialization: eliminates zero-multiplies in inner loop
            switch (bq->svf_type) {
                case FILTER_LOWPASS:
                    for (uint32_t i = 0; i < count; i++) {
                        float in = *sp;
                        float v3 = in - ic2eq;
                        float v1 = a1 * ic1eq + a2 * v3;
                        float v2 = ic2eq + a2 * ic1eq + a3 * v3;
                        ic1eq = 2.0f * v1 - ic1eq;
                        ic2eq = 2.0f * v2 - ic2eq;
                        *sp++ = v2;
                    }
                    break;
                // ... HIGHPASS, PEAKING, SHELF 等特化分支 ...
            }
            bq->svic1eq = ic1eq;
            bq->svic2eq = ic2eq;
        }
    }
}
```

**逐段解析**：

- `__restrict` 关键字告诉编译器 biquads 和 samples 指针不重叠，允许更激进的优化（避免别名检查的 load/store 屏障）。
- 外层循环按频段遍历（而非按样本），这样每个频段一次性处理整个块，系数在寄存器中复用，减少内存访问——这是经典的 **block-by-filter** 处理顺序，比 sample-by-sample 快得多。
- SVF 核心是两步梯形积分：先算 `v1`（一阶积分器输出）和 `v2`（二阶积分器输出），再用 `2*v - ic` 做状态更新（这是陷阱积分器的特征——状态更新用 2 倍中间值减旧状态，保证数值稳定）。
- `switch (bq->svf_type)` 针对每种滤波器类型生成**特化内循环**。低通直接输出 `v2`（二阶积分器输出即低通），省去 `m0*in + m1*v1 + m2*v2` 的通用混合——这种特化消除了零乘法，在实时音频中每条乘法都是宝贵的 CPU 周期。
- `ic1eq`/`ic2eq` 作为局部变量在循环内累积，循环结束后才写回结构体，避免每个样本都做内存写。

### 6.6 BS2B 耳机串扰消除（crossfeed.c）

BS2B（Bauer Stereophonic-to-Binaural）通过模拟扬声器听感来减少耳机的不自然分离度。DSPi 的实现采用互补滤波器设计：

```c
void crossfeed_compute_coefficients(CrossfeedState *state, const CrossfeedConfig *config, float sample_rate) {
    // ...
    // Compute crossfeed gain G using complementary constraint
    // feed_db is the level difference: 20*log10(direct_dc / cross_dc)
    // With complementary constraint: direct_dc + cross_dc = 1
    float level_ratio = powf(10.0f, feed_db / 20.0f);
    float G = 1.0f / (1.0f + level_ratio);

    // Lowpass filter (crossfeed path) - single pole IIR
    float x = expf(-2.0f * 3.1415926535f * fc / sample_rate);
    float lp_a0_f = G * (1.0f - x);
    float lp_b1_f = x;

    // All-pass filter for Interaural Time Delay (ITD)
    float ap_a_f;
    if (config->itd_enabled) {
        float lp_delay_sec = x / ((1.0f - x) * sample_rate);
        float remaining_sec = CROSSFEED_ITD_SEC - lp_delay_sec;
        if (remaining_sec > 0.0f) {
            float D = remaining_sec * sample_rate;
            ap_a_f = (1.0f - D) / (1.0f + D);
        } else {
            ap_a_f = 1.0f;
        }
    } else {
        ap_a_f = 1.0f;
    }
```

**逐段解析**：

- **互补约束** `direct_dc + cross_dc = 1` 是这个实现的精髓。传统 BS2B 直接衰减直通信号再叠加串扰信号，而互补设计让直通路径 = `输入 - 低通(输入)`，这样单声道信号（左右相同）在 DC 处增益恒为 1（`direct + cross = 1`），不会因为开启串扰而改变整体音量。
- `level_ratio = 10^(feed_db/20)` 计算直通与串扰的线性幅度比，`G = 1/(1+ratio)` 是串扰路径的 DC 增益。以默认 4.5 dB 为例：ratio≈1.679，G≈0.373，直通≈0.627。
- 低通滤波器 `H(z) = G*(1-x)/(1 - x*z^-1)` 模拟头部阴影效应（ILD，双耳声级差），`x = exp(-2π·fc/Fs)` 是单极 IIR 的反馈系数。
- **ITD（双耳时间差）** 通过一阶全通滤波器实现。注释中的数学非常精妙：低通滤波器本身在 DC 处就有群延迟 `τ_lp = x/((1-x)·Fs)`，ITD 目标是 220μs（人头部尺寸的声学延迟），全通滤波器只需补偿差值。全通系数 `a = (1-D)/(1+D)` 中 D 是剩余延迟样本数，这保证了总延迟精确等于目标 ITD。
- 当 `ap_a = 1.0` 时全通退化为直通（ITD 关闭），优雅地处理了开关切换。

处理函数（RP2350 浮点版）展示了互补混合的实际运算：

```c
DSP_TIME_CRITICAL
void crossfeed_process_stereo(CrossfeedState *state, float *left, float *right) {
    float in_L = *left;
    float in_R = *right;

    // Lowpass filter both channels
    float lp_out_L = state->lp_a0 * in_L + state->lp_b1 * state->lp_state_L;
    float lp_out_R = state->lp_a0 * in_R + state->lp_b1 * state->lp_state_R;
    state->lp_state_L = lp_out_L;
    state->lp_state_R = lp_out_R;

    // All-pass filter on crossfeed signals for ITD
    float ap_out_L = state->ap_a * lp_out_L + state->ap_state_L;
    state->ap_state_L = lp_out_L - state->ap_a * ap_out_L;
    float ap_out_R = state->ap_a * lp_out_R + state->ap_state_R;
    state->ap_state_R = lp_out_R - state->ap_a * ap_out_R;

    // Complementary mixing with ITD:
    *left  = (in_L - lp_out_L) + ap_out_R;
    *right = (in_R - lp_out_R) + ap_out_L;
}
```

**逐段解析**：

- 低通滤波左右声道，得到各自的串扰信号（模拟对侧声音经头部衰减后的低频成分）。
- 全通滤波对串扰信号施加 ITD 延迟，模拟声音绕过头部到达对侧耳朵的时间差。全通采用转置直接 II 型（TDF2），`y = a*x + s; s = x - a*y` 保证数值稳定。
- 最终混合：左耳 = （左直通 - 左低通）+ 右串扰延迟。直通路径 `in - lp` 是互补项——它去除了左声道中将被串扰替代的低频部分，避免低频叠加导致增益超标。右串扰 `ap_out_R` 是经延迟的右声道低频，模拟右扬声器声音传到左耳。

### 6.7 PDM 超低音 Sigma-Delta 调制器（pdm_generator.c）

这是项目中最具硬件特色的模块。它用 PIO 硬件 + 软件 Sigma-Delta 调制，直接从 GPIO 输出 1-bit PDM 码流驱动有源低音炮，无需额外 DAC。

#### 6.7.1 硬件初始化

```c
static const uint16_t pio_pdm_instr[] = { 0x6001 }; // 0: out pins, 1

void pdm_setup_hw(uint8_t pin) {
    pdm_current_pin = pin;

    // Pre-fill buffer with 50% duty cycle silence before DMA starts
    for (int i = 0; i < PDM_DMA_BUFFER_SIZE; i++) {
        pdm_dma_buffer[i] = 0xAAAAAAAA;
    }

    pdm_pio_offset = pio_add_program(PDM_PIO, &pio_pdm_program);
    pio_sm_config c = pio_get_default_sm_config();
    sm_config_set_wrap(&c, pdm_pio_offset, pdm_pio_offset + (pio_pdm_program.length - 1));
    sm_config_set_out_pins(&c, pdm_current_pin, 1);
    sm_config_set_out_shift(&c, true, true, 32);
    sm_config_set_fifo_join(&c, PIO_FIFO_JOIN_TX);

    pio_gpio_init(PDM_PIO, pdm_current_pin);
    pio_sm_set_consecutive_pindirs(PDM_PIO, PDM_SM, pdm_current_pin, 1, true);
    pio_sm_init(PDM_PIO, PDM_SM, pdm_pio_offset, &c);

    pdm_update_clock(48000);
    pio_sm_set_enabled(PDM_PIO, PDM_SM, true);

    pdm_dma_chan = dma_claim_unused_channel(true);
    dma_channel_config dmac = dma_channel_get_default_config(pdm_dma_chan);
    channel_config_set_transfer_data_size(&dmac, DMA_SIZE_32);
    channel_config_set_read_increment(&dmac, true);
    channel_config_set_write_increment(&dmac, false);
    channel_config_set_dreq(&dmac, pio_get_dreq(PDM_PIO, PDM_SM, true));
    channel_config_set_ring(&dmac, false, PDM_DMA_RING_BITS);
    dma_channel_configure(pdm_dma_chan, &dmac, &PDM_PIO->txf[PDM_SM], pdm_dma_buffer, 0xFFFFFFFF, true);
}
```

**逐段解析**：

- `pio_pdm_instr[] = { 0x6001 }` 是一条 PIO 指令 `out pins, 1`——从 OSR（输出移位寄存器）移出 1 位到引脚。整个 PDM 输出只需**一条指令**！PIO 每个时钟周期执行一次，自动从 FIFO 拉取新数据，这就是 RP2040 PIO 的威力。
- `0xAAAAAAAA` 预填充 DMA 缓冲为 50% 占空比（交替的 1010...），这是 PDM 的"静音"码型——平均值为零，无 DC 偏移。
- `sm_config_set_out_shift(&c, true, true, 32)`：右移（`true`）、自动重装（`true`）、32 位阈值。PIO 在移空 32 位 OSR 后自动从 TX FIFO 拉取下一个 word。
- `PIO_FIFO_JOIN_TX` 合并 RX 和 TX FIFO 为 8 深的 TX FIFO，给 DMA 更多缓冲余量。
- DMA 配置为环形缓冲（`channel_config_set_ring`，13 位 = 2048×4 字节），`0xFFFFFFFF` 传输计数让 DMA 永远循环——软件通过更新缓冲区内容而非重启 DMA 来喂送新数据。

#### 6.7.2 噪声整形抖动

PDM 调制质量的关键在于噪声整形——把量化噪声推到可听频带之外：

```c
#define NS_B0  15778   // 0.9630
#define NS_B1 -31556   // -1.9260
#define NS_B2  15778   // 0.9630
#define NS_A1  31531   // 1.9245 (negated for subtraction)
#define NS_A2  15580   // 0.9511

static inline int32_t noise_shaped_dither(noise_shaper_t *ns, int32_t raw_dither, int32_t quant_error) {
    // Accumulate quantization error with decay (LP filter on error)
    ns->err_acc = ((ns->err_acc * 248) >> 8) + (quant_error >> 6);

    // Combine raw dither with error feedback
    int32_t input = raw_dither - ns->err_acc;

    // Apply 2nd-order Butterworth HP filter
    int32_t output = (NS_B0 * input + NS_B1 * ns->x1 + NS_B2 * ns->x2
                    + NS_A1 * ns->y1 - NS_A2 * ns->y2) >> 14;

    ns->x2 = ns->x1;
    ns->x1 = input;
    ns->y2 = ns->y1;
    ns->y1 = output;
    return output;
}
```

**逐段解析**：

- 系数 `NS_B0/B1/B2/A1/A2` 是 8 kHz 截止频率的**二阶巴特沃斯高通**，工作在 384 kHz（48 kHz × 8 倍过采样）。它将抖动噪声推向 8 kHz 以上（人耳对高频噪声不敏感，且后续模拟低通会滤除）。
- `err_acc` 是误差反馈累加器，`* 248 >> 8` 等于乘 0.97（指数衰减），把前一次的量化误差反馈回输入，形成噪声整形的一阶反馈环。
- 高通滤波用 Q14 定点（`>> 14` 归一化），系数在 ±2 范围内适合 16 位存储。

#### 6.7.3 二阶 Sigma-Delta 调制核心

```c
// 256x Oversampling with 2nd-order sigma-delta modulator
for (int chunk = 0; chunk < 8; chunk++) {
    int32_t raw_rand = (int32_t)(fast_rand() & PDM_DITHER_MASK) - (PDM_DITHER_MASK >> 1);
    int32_t dither = noise_shaped_dither(&ns, raw_rand, local_pdm_err2 >> 8);

    uint32_t pdm_word = 0;
    for (int k = 0; k < 32; k++) {
        int32_t fb_val = ((local_pdm_err2 + dither) >= 0) ? 65535 : 0;
        if ((local_pdm_err2 + dither) >= 0) pdm_word |= (1u << (31 - k));

        local_pdm_err += (target - fb_val);
        local_pdm_err2 += (local_pdm_err - fb_val);
    }

    pdm_dma_buffer[local_pdm_write] = pdm_word;
    local_pdm_write = (local_pdm_write + 1) & (PDM_DMA_BUFFER_SIZE - 1);
}
```

**逐段解析**：

- 外层 `chunk` 循环 8 次，每次生成 32 位 PDM 数据，共 256 位 = 256 倍过采样（48 kHz 音频 → 12.288 MHz PDM 码率）。
- `fast_rand()` 是 xorshift 伪随机数生成器（`^<<13, ^>>17, ^<<5`），产生 TPDF 三角概率密度抖动，消除量化失真的谐波结构。
- 内层 `k` 循环逐位调制：比较二阶误差积分器 `local_pdm_err2` 与 0，输出 1（`65535`）或 0。`65535` 是 Q16 满量程，对应 PDM 高电平。
- **二阶 Sigma-Delta**：`local_pdm_err += (target - fb_val)` 是一阶积分器，`local_pdm_err2 += (local_pdm_err - fb_val)` 是二阶积分器。两层积分使量化噪声以 12 dB/oct 上升（二阶特性），配合噪声整形高通，可听带内信噪比远超 1-bit 量化的理论极限。
- `pdm_word |= (1u << (31 - k))` 按位拼装 32 位 PDM 字，MSB 先发（与 PIO 右移配置匹配）。

#### 6.7.4 淡入淡出与防爆音

```c
// Audio fade-in
if (fade_in_pos < PDM_FADE_IN_SAMPLES) {
    pcm_val = (pcm_val * (int32_t)fade_in_pos) >> PDM_FADE_IN_SHIFT;
    fade_in_pos++;
}
fade_base_pcm = pcm_val;
```

- `PDM_FADE_IN_SAMPLES = 1024`（约 21ms @ 48kHz）的线性淡入，从零渐变到满幅度，消除 PDM 启停时的爆音。淡出同理反向操作。`>> PDM_FADE_IN_SHIFT`（即 `>> 10`）是定点比例缩放，比浮点乘法快。

#### 6.7.5 Core 1 入口与模式分发

```c
void pdm_core1_entry() {
#if PICO_RP2350
    // Enable flush-to-zero and default-NaN on Core 1
    {
        uint32_t fpscr;
        __asm__ volatile("vmrs %0, fpscr" : "=r"(fpscr));
        fpscr |= (1 << 24) | (1 << 25);  // FZ + DN bits
        __asm__ volatile("vmsr fpscr, %0" : : "r"(fpscr));
    }
#endif

    multicore_lockout_victim_init();

    while (1) {
        switch (core1_mode) {
            case CORE1_MODE_PDM:
                pdm_processing_loop();
                break;
            case CORE1_MODE_EQ_WORKER:
                eq_worker_loop();
                break;
            default:
                global_status.cpu1_load = 0;
                __wfe();
                break;
        }
    }
}
```

**逐段解析**：

- RP2350 上通过内联汇编设置 FPSCR 的 FZ（Flush-to-Zero）和 DN（Default-NaN）位。这把非规格化浮点数（denormal）强制清零——音频滤波器在静音时状态会衰减到极小值，非规格化数会触发 CPU 微码慢路径，性能下降数十倍。这是实时浮点音频处理的必备优化。
- `multicore_lockout_victim_init()` 注册 Core 1 为闪存操作"受害者"：当 Core 0 需要擦写 Flash 时（XIP 必须暂停），能安全地把 Core 1 停在 RAM 中，避免 Flash 访问冲突。
- `__wfe()`（Wait For Event）让空闲的 Core 1 进入低功耗等待，由 Core 0 的 `__sev()`（Send Event）唤醒。

### 6.8 双核 EQ Worker 并行处理

Core 1 在 EQ Worker 模式下与 Core 0 并行处理输出通道：

```c
static void __not_in_flash_func(eq_worker_loop)() {
    c1eq_load_q8 = 0;

    while (core1_mode == CORE1_MODE_EQ_WORKER) {
        while (!core1_eq_work.work_ready) {
            if (core1_mode != CORE1_MODE_EQ_WORKER) return;
            __wfe();
        }
        __dmb();

        // Process EQ + gain for outputs assigned to Core 1
        extern MatrixMixer matrix_mixer;
        for (int out = CORE1_EQ_FIRST_OUTPUT; out <= CORE1_EQ_LAST_OUTPUT; out++) {
            if (!matrix_mixer.outputs[out].enabled) continue;

            // Output EQ
            if (!matrix_mixer.outputs[out].mute) {
                uint8_t eq_ch = CH_OUT_1 + out;
                if (!channel_bypassed[eq_ch]) {
                    dsp_process_channel_block(filters[eq_ch], buf_out[out], sample_count, eq_ch);
                }
            }

            // Combined gain + volume — per-sample linear ramp
            float matrix_gain = matrix_mixer.outputs[out].gain_linear;
            float gain_start = matrix_mixer.outputs[out].mute ? 0.0f
                               : matrix_gain * vol_mul_start;
            float gain_step  = matrix_mixer.outputs[out].mute ? 0.0f
                               : matrix_gain * vol_mul_step;
            // ... apply gain with ramp ...
        }

        // Delay for Core 1 outputs
        if (any_delay_active) {
            for (int out = CORE1_EQ_FIRST_OUTPUT; out <= CORE1_EQ_LAST_OUTPUT; out++) {
                int32_t dly = channel_delay_samples[out];
                if (dly <= 0) continue;
                float *dst = buf_out[out];
                float *dline = delay_lines[out];
                uint32_t widx = core1_eq_work.delay_write_idx;
                for (uint32_t i = 0; i < sample_count; i++) {
                    dline[widx] = dst[i];
                    dst[i] = dline[(widx - dly) & MAX_DELAY_MASK];
                    widx = (widx + 1) & MAX_DELAY_MASK;
                }
            }
        }

        // Signal completion to Core 0
        core1_eq_work.work_ready = false;
        __dmb();
        core1_eq_work.work_done = true;
        __sev();
    }
}
```

**逐段解析**：

- `__not_in_flash_func` 将整个函数放入 RAM，因为 Flash XIP 在高采样率下会成为瓶颈（96 kHz 时每样本预算仅 ~3μs）。
- `__dmb()`（Data Memory Barrier）确保 Core 0 写入的工作描述符（`work_ready`、缓冲区指针、采样数）对 Core 1 全部可见，避免缓存一致性问题导致的竞态。
- **增益斜坡**（`vol_mul_start`/`vol_mul_step`）实现音量的无爆音渐变：每个样本乘以一个线性递增/递减的系数，`step==0` 时退化为常数增益快路径。这比直接跳变音量优雅得多。
- **延迟线**用环形缓冲实现：`dline[widx] = dst[i]` 写入当前样本，`dline[(widx - dly) & MAX_DELAY_MASK]` 读取 dly 个样本前的数据。`& MAX_DELAY_MASK` 是 2 的幂取模优化（比 `%` 快）。这条延迟线用于扬声器/低音炮时间对齐。
- 完成后 `__dmb()` + `work_done = true` + `__sev()` 通知 Core 0，形成生产者-消费者握手。

---

## 七、USB 控制协议（API 使用指南）

DSPi 通过 USB Vendor 接口（WinUSB/WCID）暴露控制协议。主机应用通过 Vendor Request 与固件交互：

| 请求码 | 功能 | 说明 |
|---|---|---|
| `0x50` | GET_STATUS | 读取系统遥测（CPU 负载、峰值、剪辑标志） |
| `0x7C` | SET_OUTPUT_PIN | 运行时重分配输出引脚 |
| `0xC2` | SET_I2S_BCK_PIN | 设置 I2S 位时钟引脚 |
| `0xC6` | SET_MCK_PIN | 设置主时钟引脚 |
| `0xE4` | SET_SPDIF_RX_PIN | 设置 S/PDIF 输入引脚 |

### cURL 示例（通过 USB 工具）

由于是 USB Vendor 请求而非 HTTP，需使用 `curl` 配合 USB 工具或专用控制台应用。以下为概念示例：

```bash
# 列出 USB 设备（Linux）
lsusb | grep "Weeb Labs"

# 使用 DSPi Console 应用控制（推荐方式）
# 设置输出槽 0 为 I2S 模式
./dspi_console --output 0 --type i2s

# 设置主音量为 -10 dB
./dspi_console --master-volume -10

# 加载预设槽 3
./dspi_console --preset load 3
```

---

## 八、编译与部署

### 8.1 环境搭建

| 组件 | 版本要求 |
|---|---|
| 操作系统 | Linux / macOS / Windows |
| 工具链 | arm-none-eabi-gcc 10.3+ |
| 构建系统 | CMake 3.13+ |
| SDK | Pico SDK 2.x（含 pico-extras） |
| 依赖 | git（子模块拉取 pico-sdk） |

### 8.2 从源码编译

```bash
git clone https://github.com/WeebLabs/DSPi.git
cd DSPi/firmware
git submodule update --init --recursive

mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
```

编译产物为 `DSPi.uf2`。

### 8.3 烧录步骤

1. 按住 Pico 的 **BOOTSEL** 按钮插入 USB
2. 板子会枚举为 `RPI-RP2` U 盘
3. 将 `DSPi.uf2` 拖入该 U 盘
4. Pico 自动重启，枚举为 "Weeb Labs DSPi" 音频设备
5. 下载并运行 DSPi Console 应用控制 DSP 参数

### 8.4 使用方法

烧录后，Pico 即可作为系统声卡使用。在系统声音设置中选择 "Weeb Labs DSPi" 作为输出设备。通过 DSPi Console 应用可以实时调节：

- 参数均衡（每通道 10 段）
- 矩阵混音器（输入到输出的路由）
- 时间对齐（每输出延迟）
- 响度补偿与耳机串扰
- 主音量与每通道增益
- 10 个预设槽位的保存/加载

---

## 九、项目亮点与适用场景

### 技术亮点

- **单指令 PIO PDM 输出**：用一条 PIO 指令 + DMA 环形缓冲实现 256 倍过采样 PDM，无需额外 DAC 芯片
- **混合 SVF/biquad 架构**：低频用 SVF 保证数值精度，高频用 biquad 保证效率，自动切换
- **Q28 定点 + 手写汇编**：RP2040 无 FPU，用定点运算和 ARM 汇编榨取实时性能
- **互补滤波器串扰消除**：BS2B 用互补设计保证单声道增益恒定，避免传统方案的音量漂移
- **无爆音采样率切换**：静音排空 + 系数重算 + 同步重启的三阶段原子切换
- **双核流水线并行**：Core 0/1 分工处理 EQ + 增益 + 延迟，最大化吞吐

### 适用场景

- HiFi 有源音箱数字分频（2-way/3-way）
- 耳机放大器参数均衡与串扰消除
- 家庭影院低音管理与时间对齐
- 录音棚监听扬声器房间校正
- 便携 USB 音频处理器（DIY 项目）
- 嵌入式音频 DSP 教学参考实现

---

## 十、总结

DSPi 是我近期在 GitHub 上发现的最令人印象深刻的 RP2040 音频项目之一。它不是简单的"USB 声卡 Demo"，而是一套**生产级的数字音频处理器**——从 USB Audio Class 设备栈、双核 DSP 流水线、PIO 硬件 PDM 输出，到混合 SVF/biquad 滤波架构、无爆音采样率切换、闪存预设系统，每个环节都体现了深厚的音频工程功底。

最让我欣赏的是它在有限硬件上的工程取舍：RP2040 用 Q28 定点 + 手写汇编弥补无 FPU 的短板，RP2350 用硬件浮点 + SVF 架构追求更高精度；PDM 输出仅用一条 PIO 指令就实现了 256 倍过采样 Sigma-Delta 调制，把微控制器的成本压缩到极致。代码注释极其详尽——几乎每个非平凡的设计决策都有原理解释，对学习嵌入式音频 DSP 的开发者来说是难得的教材级开源项目。如果你对实时音频处理、PIO 编程或定点 DSP 优化感兴趣，DSPi 绝对值得深入研读。

---

> 📝 作者：蔡浩宇（jun-chy）
>
> 📅 日期：2026-06-28
>
> 🔗 项目地址：https://github.com/WeebLabs/DSPi
