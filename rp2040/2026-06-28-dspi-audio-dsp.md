# DSPi：把树莓派 Pico 变成专业级 USB 音频 DSP 处理器

> 项目地址：<https://github.com/WeebLabs/DSPi>
> 仓库作者：WeebLabs
> 许可证：GPL-3.0
> 截至撰稿日 Star 数：1010+（持续增长中，最近更新于 2026 年 6 月）

## 一、项目简介

**DSPi** 是一个把 Raspberry Pi Pico（RP2040）或 Pico 2（RP2350）变成一台「专业级数字音频处理器」的开源固件项目。它把几十块钱的开发板伪装成一个 USB 声卡，并在板载的 DSP 引擎里完成房间校正、有源分频、参数均衡（PEQ）、时间对齐、响度补偿、耳机交叉馈送等一系列专业音频处理。

作者给它的定位是：让 RP2040 / RP2350 成为「不到一杯咖啡钱的音频瑞士军刀」。它跨 macOS、Windows、Linux、iOS 即插即用，支持 16/24-bit PCM、44.1 / 48 / 96 kHz 采样率，可输出最多 4 路立体声 S/PDIF 或 I2S（共 8 通道，RP2350），并带有一路独立 PDM 超低音输出。

**核心特点：**

- USB 音频接口（UAC），免驱即插即用
- 高达 10 段/通道参数均衡，RP2350 上共 110 段滤波器
- 矩阵混音器（2 进 × 9 出），每个交叉点独立增益与反相
- BS2B 耳机交叉馈送（含耳间时延 ITD）
- 基于 ISO 226:2003 的响度补偿
- RMS 上行压限器（Volume Leveller）
- 双核 DSP：EQ 处理拆分到两个核并行执行
- 10 槽位预设系统，可热保存/加载
- 全部输出引脚可在运行时重映射，适配自定义 PCB

**典型应用场景：** PC/Mac 外接 DSP 音箱控制器、有源分频音箱、耳机放大器的前级 DSP、超低音管理、低成本房间校正处理器。

## 二、技术栈分析

| 维度 | 说明 |
|------|------|
| 芯片平台 | RP2040（Pico）/ RP2350（Pico 2） |
| 框架 | Pico SDK + pico-extras（裸机，无 RTOS） |
| USB 协议栈 | TinyUSB（UAC 音频类 + WinUSB 厂商接口） |
| 构建系统 | CMake |
| 数学引擎 | RP2040：Q28 定点 + 手写 ARM 汇编（`dsp_process_rp2040.S`）；RP2350：硬件单精度浮点 FPU |
| 时钟 | 307.2 MHz（RP2040 为超频 + 微调电压） |
| 输出 | 24-bit S/PDIF / I2S（PIO 实现）+ PDM 超低音（二阶 ΔΣ 调制） |
| 语言 | C + ARM 汇编 |

一个值得关注的工程亮点是：**同一套源码用条件编译同时支持 RP2040 与 RP2350 两块芯片**。RP2040 没有硬件浮点单元，作者用 Q28 定点 + 手写汇编来保证实时性；RP2350 有硬件 FPU，则切到浮点 + SVF（状态变量滤波器）架构以获得更优的低频精度。

## 三、核心代码片段与解析

下面从仓库中提取几段真实的关键代码，逐段解析其实现原理。源码位于 `firmware/DSPi/` 目录下。

### 3.1 Q28 定点快速乘法（RP2040 实时性的基石）

RP2040 没有硬件浮点，DSPi 把所有系数用 Q28 定点表示（`FILTER_SHIFT = 28`）。`dsp_pipeline.c` 里实现了一个 32×32→32 的快速定点乘法 `fast_mul_q28`：

```c
// 文件：firmware/DSPi/dsp_pipeline.c

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

**代码解析：**

- Q28 定点用一个 32 位整数表示一个范围 [-8, 8) 的小数，比例因子为 2^28。两个 Q28 数相乘后结果是 Q56，需要右移 28 位才能回到 Q28。
- 标准做法需要 64 位乘法（`((int64_t)a * b) >> 28`），但 ARM Cortex-M0+ 上 64 位运算较慢。
- 这里把 32 位数拆成「高 16 位 + 低 16 位」做三组 16×16 乘法（`ah*bh`、`ah*bl`、`al*bh`），再用移位拼回 Q28，避免 64 位中间结果，在 M0+ 上更快。
- `high << 4`：`ah*bh` 是 Q32 结果（两个 Q16 相乘），左移 4 位变为 Q28。`(mid1+mid2) >> 12`：交叉项是 Q32，右移 12 位变 Q20… 这里通过精心设计的移位把各项对齐到 Q28。
- `DSP_TIME_CRITICAL` 宏把函数放进时间关键段，配合 `-O3` 与链接脚本里的 RAM 运行段（`copy_to_ram`）保证零 Flash 等待。

### 3.2 双精度架构的均衡器系数计算（RBJ Cookbook + SVF）

参数均衡是整个 DSP 管线的核心。`dsp_compute_coefficients()` 根据滤波器类型（峰值、低/高架、低/高通、陷波、全通）计算系数。RP2350 上低频段自动切换到 SVF 拓扑以提升精度：

```c
// 文件：firmware/DSPi/dsp_pipeline.c

void dsp_compute_coefficients(EqParamPacket *p, Biquad *bq, float sample_rate) {
    bool user_bypass = (p->bypass == 1);

    if (user_bypass || is_filter_flat(p) || sample_rate == 0) {
        bq->bypass = true;
        // ... 系数置为直通 ...
        return;
    }

    bq->bypass = false;

    // 输入校验：Q 限制在 [0.1, 20]，频率限制在 [10Hz, 0.45*Fs]
    if (p->Q < 0.1f) p->Q = 0.1f;
    if (p->Q > 20.0f) p->Q = 20.0f;
    if (p->freq < 10.0f) p->freq = 10.0f;
    if (p->freq > sample_rate * 0.45f) p->freq = sample_rate * 0.45f;

    float A = powf(10.0f, p->gain_db / 40.0f);

#if PICO_RP2350
    // SVF / biquad 分频点决策：低于 Fs/7.5 用 SVF
    bool was_svf = bq->use_svf;
    bq->use_svf = (p->freq < (sample_rate / 7.5f));
    if (was_svf != bq->use_svf) {
        bq->s1 = 0.0f; bq->s2 = 0.0f;
        bq->svic1eq = 0.0f; bq->svic2eq = 0.0f;
    }

    if (bq->use_svf) {
        // SVF 系数（Simper / Cytomic "SvfLinearTrapAllOutputs", 2021）
        float g = tanf(3.1415926535f * p->freq / sample_rate);
        float k = 1.0f / p->Q;
        // ... 根据类型调整 g 和 k ...
        float sva1_f = 1.0f / (1.0f + g * (g + k));
        float sva2_f = g * sva1_f;
        float sva3_f = g * sva2_f;
        // ... 按类型计算混合系数 svm0/svm1/svm2 ...
        return;
    }
#endif

    // RBJ Audio EQ Cookbook 双二阶（biquad）系数
    float omega = 2.0f * 3.1415926535f * p->freq / sample_rate;
    float sn = sinf(omega); float cs = cosf(omega);
    float alpha = sn / (2.0f * p->Q);
    // ... 按 FILTER_PEAKING / LOWSHELF / HIGHSHELF 等计算 b0..a2 ...

#if PICO_RP2350
    // 浮点存储
    float inv_a0 = 1.0f / a0_f;
    bq->b0 = b0_f * inv_a0;  // ...
#else
    // Q28 定点存储
    float scale = (float)(1LL << FILTER_SHIFT);
    bq->b0 = (int32_t)((b0_f / a0_f) * scale);
    // ...
#endif
}
```

**代码解析：**

- `is_filter_flat()` 判断该频段是否「实际上没效果」（增益 < 0.01dB 的峰值/架式滤波器），直接 bypass 以省 CPU。这是个很实用的优化：用户即便留了 10 段空 EQ，也只对真正起作用的频段做计算。
- RP2350 分支用了一个**混合 SVF / biquad 架构**：当中心频率低于 `Fs/7.5` 时用 SVF（State Variable Filter），高频段仍用传统 biquad。原因是经典 biquad 在低频（相对采样率）时系数对量化误差极敏感，会出现数值不稳定；而 SVF 在低频数值特性更好。作者引用了 Andrew Simper（Cytomic）2021 年的 `SvfLinearTrapAllOutputs` 公式。
- `A = 10^(gain_db/40)`：注意分母是 40 而不是 20，因为增益在系数里以 `A^2` 形式出现（`b0 = 1 + alpha*A` 等），等价于 `10^(gain_db/20)`。
- 同一函数在 RP2040 上把系数量化到 Q28（`* scale`），在 RP2350 上存浮点。这种「算法一致、存储格式按平台切换」的设计让两块芯片的滤波器响应完全一致。

### 3.3 分块处理与按类型特化（热循环优化）

实时音频处理对每样本开销极其敏感。`dsp_process_channel_block()` 对一整块样本做处理，并**针对每种滤波器类型展开独立内循环**，消除零乘法：

```c
// 文件：firmware/DSPi/dsp_pipeline.c (RP2350 分支)

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

            // 按类型特化：消除内循环里的零乘法
            switch (bq->svf_type) {
                case FILTER_LOWPASS:
                    for (uint32_t i = 0; i < count; i++) {
                        float in = *sp;
                        float v3 = in - ic2eq;
                        float v1 = a1 * ic1eq + a2 * v3;
                        float v2 = ic2eq + a2 * ic1eq + a3 * v3;
                        ic1eq = 2.0f * v1 - ic1eq;
                        ic2eq = 2.0f * v2 - ic2eq;
                        *sp++ = v2;            // 低通输出 = v2
                    }
                    break;
                case FILTER_HIGHPASS:
                    for (uint32_t i = 0; i < count; i++) {
                        float in = *sp;
                        float v3 = in - ic2eq;
                        float v1 = a1 * ic1eq + a2 * v3;
                        float v2 = ic2eq + a2 * ic1eq + a3 * v3;
                        ic1eq = 2.0f * v1 - ic1eq;
                        ic2eq = 2.0f * v2 - ic2eq;
                        *sp++ = in + m1 * v1 - v2;   // 高通 = in - 低通
                    }
                    break;
                // ... PEAKING/NOTCH/ALLPASS、LOWSHELF/HIGHSHELF 各自展开 ...
            }
            bq->svic1eq = ic1eq;
            bq->svic2eq = ic2eq;
        } else {
            // 单精度 TDF-II biquad
            float b0 = bq->b0, b1 = bq->b1, b2 = bq->b2;
            float a1 = bq->a1, a2 = bq->a2;
            float s1 = bq->s1, s2 = bq->s2;
            float *sp = samples;
            for (uint32_t i = 0; i < count; i++) {
                float in = *sp;
                float out = b0 * in + s1;
                s1 = b1 * in - a1 * out + s2;
                s2 = b2 * in - a2 * out;
                *sp++ = out;
            }
            bq->s1 = s1;  bq->s2 = s2;
        }
    }
}
```

**代码解析：**

- **逐频段串行处理**：外层循环遍历每个 EQ 频段，对整块样本依次做该频段的滤波。这比「逐样本遍历所有频段」对缓存更友好——同一块数据在 SRAM 里被连续访问多次。
- **`__restrict` 关键字**告诉编译器 `biquads` 与 `samples` 不别名，允许更激进的优化。
- **按类型特化内循环**是关键优化：SVF 的通用输出公式是 `out = m0*in + m1*v1 + m2*v2`，但对低通 `m0=0, m1=0, m2=1`（即 `out=v2`），高通 `out = in - v2`。把这些零乘法和常数项在编译期消掉，每个样本少 1~2 次乘加。在 96kHz、11 通道、每通道 10 段的满载下，这种优化是能否跑得动的关键。
- RP2040 上同样的函数是**用 ARM 汇编**（`dsp_process_rp2040.S`）实现的，声明为 `extern`，因为 Cortex-M0+ 上手写汇编排程比 C 更可控。
- 状态变量 `ic1eq/ic2eq` 在块处理结束后才回写，循环内用本地寄存器变量，减少 SRAM 访问。

### 3.4 BS2B 耳机交叉馈送（互补滤波 + 耳间时延）

耳机听音时左右声道完全分离，缺乏音箱在房间里的串扰，听起来「头中效应」明显。`crossfeed.c` 实现了 BS2B（Bauer Stereophonic-to-Binaural）交叉馈送：

```c
// 文件：firmware/DSPi/crossfeed.c

// 预设：{截止频率 Hz, 馈送电平 dB}
static const float presets[][2] = {
    { 700.0f,  4.5f },  // Default - 平衡，最常用
    { 700.0f,  6.0f },  // Chu Moy - 更强空间感
    { 650.0f,  9.5f },  // Jan Meier - 细腻自然
};

void crossfeed_compute_coefficients(CrossfeedState *state, const CrossfeedConfig *config, float sample_rate) {
    // ... 取截止频率 fc 和馈送电平 feed_db ...

    // 用互补约束计算交叉馈送增益 G：
    //   direct_dc / cross_dc = 10^(feed_db/20)
    //   cross_dc = 1 / (1 + level_ratio) = G
    //   direct_dc = 1 - G
    float level_ratio = powf(10.0f, feed_db / 20.0f);
    float G = 1.0f / (1.0f + level_ratio);

    // 交叉路径低通（单极 IIR）：H(z) = G*(1-x) / (1 - x*z^-1)
    float x = expf(-2.0f * 3.1415926535f * fc / sample_rate);
    float lp_a0_f = G * (1.0f - x);
    float lp_b1_f = x;

    // 耳间时延（ITD）：低通已引入部分相位延迟，剩余用一阶全通补足
    float ap_a_f;
    if (config->itd_enabled) {
        float lp_delay_sec = x / ((1.0f - x) * sample_rate);
        float remaining_sec = CROSSFEED_ITD_SEC - lp_delay_sec;  // ~220µs
        if (remaining_sec > 0.0f) {
            float D = remaining_sec * sample_rate;
            ap_a_f = (1.0f - D) / (1.0f + D);
        } else {
            ap_a_f = 1.0f;
        }
    } else {
        ap_a_f = 1.0f;  // ITD 关闭：全通变直通
    }
    // ... 按 RP2350 存浮点 / RP2040 存 Q28 ...
}

// RP2350 浮点处理
DSP_TIME_CRITICAL
void crossfeed_process_stereo(CrossfeedState *state, float *left, float *right) {
    float in_L = *left;
    float in_R = *right;

    // 低通两路：cross = G × L(z) × input
    float lp_out_L = state->lp_a0 * in_L + state->lp_b1 * state->lp_state_L;
    float lp_out_R = state->lp_a0 * in_R + state->lp_b1 * state->lp_state_R;
    state->lp_state_L = lp_out_L;
    state->lp_state_R = lp_out_R;

    // 全通补足 ITD（转置直接 II 型）
    float ap_out_L = state->ap_a * lp_out_L + state->ap_state_L;
    state->ap_state_L = lp_out_L - state->ap_a * ap_out_L;
    float ap_out_R = state->ap_a * lp_out_R + state->ap_state_R;
    state->ap_state_R = lp_out_R - state->ap_a * ap_out_R;

    // 互补混合：direct = input - own_lowpass；output = direct + allpass(opp_lowpass)
    *left  = (in_L - lp_out_L) + ap_out_R;
    *right = (in_R - lp_out_R) + ap_out_L;
}
```

**代码解析：**

- **互补滤波设计**是这段代码的精髓。直接路径取 `input - lowpass(input)`，交叉路径取 `allpass(lowpass(对侧))`，两者相加。在直流处 `direct_dc + cross_dc = (1-G) + G = 1`，保证单声道信号增益为 1（不染色）；高频处低通输出趋于 0，硬声像的高频成分原样保留——这正是 BS2B 想要的「只串扰低频」效果。
- **`feed_db` 的物理含义**：直流通路与交叉通路在 DC 的电平差（dB）。用互补约束 `direct_dc + cross_dc = 1` 反解出 `G = 1/(1+10^(feed_db/20))`，4.5dB 时 G≈0.373、direct≈0.627。
- **耳间时延 ITD**：人耳对侧声音约有 220µs 延迟（头围 / 声速）。低通滤波器本身在 DC 已引入 `τ_lp = x/((1-x)*Fs)` 的群延迟，剩余部分用一阶全通 `H(z)=(a+z^-1)/(1+a*z^-1)` 补足，群延迟为 `(1-a)/(1+a)` 个样本。反解 `a=(1-D)/(1+D)`。700Hz@48kHz 时低通已提供约 217µs，全通只需补约 3µs。
- 全通用**转置直接 II 型（TDF-II）**实现：`y[n]=a*x[n]+s`、`s[n+1]=x[n]-a*y[n]`，数值稳定性好。
- RP2040 版本把 `lp_a0/lp_b1/ap_a` 存成 Q28，乘法走前面讲过的 `fast_mul_q28`，算法结构与浮点版完全对应。

### 3.5 ISO 226:2003 响度补偿

人耳对低频/高频的灵敏度随音量下降而降低（弗莱彻-芒森曲线）。`loudness.c` 实现了基于 ISO 226:2003 等响曲线的音量相关 EQ：

```c
// 文件：firmware/DSPi/loudness.c

// ISO 226:2003 表1：f=50Hz → αf=0.432, Lu=-15.9, Tf=44.0
//                   f=10kHz → αf=0.271, Lu=-10.7, Tf=13.9
#define ISO_50_TF   44.0f
#define ISO_50_AF   0.432f
#define ISO_50_LU   (-15.9f)

// ISO 226:2003 SPL 计算（标准公式 1-2）
static float iso226_spl(float Tf, float af, float Lu, float phon) {
    float B = 0.4f * powf(10.0f, (Tf + Lu) / 10.0f - 9.0f);
    float threshold = powf(B, af);
    float Af = 4.47e-3f * (powf(10.0f, 0.025f * phon) - 1.15f) + threshold;
    if (Af < 1e-10f) Af = 1e-10f;
    return (10.0f / af) * log10f(Af) - Lu + 94.0f;
}

// 计算某频率在给定音量下的补偿增益（dB）
static float loudness_compensation_db(float Tf, float af, float Lu,
                                       float ref_spl, float effective_phon,
                                       float intensity_pct) {
    if (effective_phon >= ref_spl) return 0.0f;  // 参考音量不补偿

    float spl_ref = iso226_spl(Tf, af, Lu, ref_spl);   // 参考音量下该频率的 SPL
    float spl_eff = iso226_spl(Tf, af, Lu, effective_phon);  // 降低后音量下的 SPL

    // 补偿 = 该频率 SPL 实际变化 − 1kHz 的均匀衰减量
    float flat_change = effective_phon - ref_spl;   // 负值（音量降低）
    float freq_change = spl_eff - spl_ref;
    float compensation = freq_change - flat_change;  // 正值 = 需要提升

    compensation *= (intensity_pct / 100.0f);  // 按强度缩放
    return compensation;
}
```

**代码解析：**

- 直接套用 ISO 226:2003 国际标准的等响曲线公式。`iso226_spl()` 算出「在给定响度级 phon 下，某频率需要多大声压级 SPL 才能听起来一样响」。
- 补偿量 = `(该频率 SPL 变化) − (1kHz 均匀衰减)`。因为整体音量下调时 1kHz 也跟着降，这部分不该被补偿；只补「低频掉得比 1kHz 更多」的那部分差值。这是个很巧妙的差分设计。
- 作者在注释里特别提到：旧固件曾把 `Lu` 符号写反、10kHz 的 `αf` 偏差约 10%，导致补偿量偏大 2.5 倍——说明这类音频数学的细节极易出错，对照标准常数非常关键。
- 实际只取 50Hz 和 10kHz 两个频率点，分别算出低架/高架补偿增益，再用前述 RBJ shelf 公式生成 biquad/SVF 系数。整个系数表按音量步进（-60~0dB）预计算并双缓冲，运行时按当前音量查表，避免实时算指数。

## 四、编译与部署方法

### 4.1 直接烧录（推荐入门）

1. 到 [Releases 页面](https://github.com/WeebLabs/DSPi/releases) 下载对应板子的 `DSPi.uf2`。
2. 按住 Pico 的 **BOOTSEL** 键插入电脑，出现 `RPI-RP2` U 盘。
3. 把 `.uf2` 拖入该盘，Pico 自动重启并枚举为 "Weeb Labs DSPi" 音频设备。
4. 运行 DSPi Console 桌面应用控制 DSP 参数。

### 4.2 从源码编译

```bash
# 1. 安装 Pico SDK 并设置环境变量 PICO_SDK_PATH
git clone https://github.com/raspberrypi/pico-sdk.git --recurse-submodules
export PICO_SDK_PATH=/path/to/pico-sdk

# 2. 克隆 DSPi（含子模块 pico-extras / tinyusb）
git clone --recurse-submodules https://github.com/WeebLabs/DSPi.git
cd DSPi/firmware

# 3. 构建（RP2040）
mkdir build && cd build
cmake ..
make -j

# 4. 产物 build/DSPi/DSPi.uf2，按 BOOTSEL 拖入烧录
```

### 4.3 硬件接线（RP2040 默认）

| 功能 | GPIO | 说明 |
|------|------|------|
| 输出槽 0（Out 1-2） | 6 | S/PDIF 或 I2S 数据 |
| 输出槽 1（Out 3-4） | 7 | S/PDIF 或 I2S 数据 |
| 超低音 PDM（Out 5） | 10 | 需 RC 低通转模拟 |
| I2S BCK | 14 | I2S 模式下位时钟 |
| I2S LRCLK | 15 | 固定为 BCK+1 |
| I2S MCK（可选） | 13 | 128×/256× Fs 主时钟 |

S/PDIF 输出需 Toshiba TOSLINK 光纤发射器或电阻分压；I2S 直接接大多数 I2S DAC；PDM 输出需电阻 + 电容做低通滤波。

## 五、项目亮点与适用场景

**工程亮点：**

- **同一套源码双平台**：用 `#if PICO_RP2350` 条件编译在定点/浮点、biquad/SVF 之间切换，算法一致性有保证。
- **手写 ARM 汇编**：RP2040 的逐样本 EQ 处理用 `dsp_process_rp2040.S` 实现，榨干 M0+ 的每一点性能。
- **分块 + 按类型特化**：热循环里消零乘法、寄存器缓存状态变量，是嵌入式实时 DSP 的典范写法。
- **互补滤波设计**：交叉馈送用 `input - lowpass` 保证直流增益恒为 1，理论严谨。
- **对照国际标准**：响度补偿严格套用 ISO 226:2003 公式常数，并在注释里记录了过去踩过的数值坑。

**适用场景：** PC/Mac 外接 DSP 音箱前级、有源分频多路功放系统、耳机放大器的前级 DSP、超低音管理与低通、低成本房间校正、嵌入式音频 DSP 算法学习与教学。

## 六、总结

DSPi 是一个把「专业音频 DSP」拉到几十块钱开发板上的优秀项目。它的价值不仅在于功能完整（USB 声卡 + 10 段 PEQ + 矩阵混音 + 交叉馈送 + 响度补偿 + 双核处理），更在于其源码是学习**嵌入式实时数字信号处理**的绝佳教材：Q28 定点乘法、biquad/SVF 双拓扑、互补滤波、分块按类型特化、ISO 226 等响曲线——每一块都对应音频 DSP 的一个核心知识点，且代码注释极为详尽。对想做音频嵌入式开发的工程师而言，这个仓库值得逐行研读。
