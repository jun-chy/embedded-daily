# 嵌入式日报（Embedded Daily）

> 每日嵌入式开源项目深度解析 · 代码级技术分析
>
> 作者：[蔡浩宇（jun-chy）](https://github.com/jun-chy)

这是一个嵌入式开源项目的深度分析与解读仓库。每天选取 GitHub 上活跃的嵌入式相关开源项目，从硬件架构、固件源码、核心算法到编译部署，进行系统性的代码级技术分析。

## 仓库理念

嵌入式开发不应该是黑盒。这个仓库的目标是：

- **拆解真实项目**：不是浅尝辄止的翻译 README，而是深入源码，提取关键代码片段并逐行解析
- **覆盖主流平台**：ESP32、STM32、RP2040、Arduino、nRF52、嵌入式 Linux 等主流 MCU 平台
- **注重实操**：每篇文章都包含编译方法、烧录步骤、API 调用示例，读完就能上手
- **追踪前沿**：关注近期活跃更新、star 增长快、有实际代码和文档的项目

## 目录结构

```
embedded-daily/
├── README.md                  # 本文件
├── esp32/                     # ESP32 / ESP8266 相关项目
│   └── 2026-06-28-*.md
├── stm32/                     # STM32 相关项目
│   └── 2026-06-29-*.md
├── arduino/                   # Arduino 相关项目
├── rp2040/                    # RP2040 / Raspberry Pi Pico 相关项目
│   └── 2026-06-28-*.md
├── nrf52/                     # Nordic nRF52 相关项目
├── linux-embedded/            # 嵌入式 Linux 相关项目
└── other/                     # 其他嵌入式项目
```

## 已发布文章

### 2026-07-01

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-01 | RP2040 | [DSPi](rp2040/2026-07-01-DSPi.md) | 1019 | RP2040/RP2350 全功能音频 DSP 固件，双平台自适应滤波（SVF/Biquad）+ BS2B 交叉馈送 + ISO 226 响度补偿 |
| 2026-07-01 | ESP32 | [OpenC6 BIOS](esp32/2026-07-01-openc6-bios.md) | 266 | ESP32-C6 开源 BIOS 固件平台，ABI 系统调用接口 + A/B OTA 防变砖 + PXE 网络启动 + LP-Core 管理引擎 |

### 2026-06-29

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-06-29 | STM32 | [tinyrtos](stm32/2026-06-29-tinyrtos.md) | 11 | 从零手写的 ARM Cortex-M 抢占式 RTOS 内核，PendSV 上下文切换 + 优先级调度 + HardFault 诊断，内核仅 9KB |
| 2026-06-29 | STM32 | [Filum](stm32/2026-06-29-filum.md) | 28 | MCU 级联邦学习框架，STM32 + LoRa 上的边缘 AI 训练，Top-k 稀疏 Q8 量化 + 高斯差分隐私 + ChaCha20 加密 |

### 2026-06-28

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-06-28 | ESP32 | [Sesame Robot](esp32/2026-06-28-sesame-robot.md) | 3,075 | 基于 ESP32 的开源四足机器人，8 舵机驱动，OLED 表情系统，WiFi 双模式 + RESTful API |
| 2026-06-28 | ESP32 | [OpenC6 BIOS](esp32/2026-06-28-openc6-bios.md) | 238 | 把 PC 主板 BIOS 架构搬进 ESP32-C6，LP-Core 管理引擎 + PXE 网络启动 + A/B OTA |
| 2026-06-28 | RP2040 | [DeskHop](rp2040/2026-06-28-deskhop.md) | 7,624 | 双 RP2040 硬件 KVM 切换器，PIO USB + DMA UART + 绝对坐标鼠标跨屏切换 |
| 2026-06-28 | RP2040 | [DSPi](rp2040/2026-06-28-dspi-audio-dsp.md) | 1,010 | RP2040/RP2350 专业级 USB 音频 DSP，参数均衡 + 矩阵混音 + 房间校正 |

### 文章技术覆盖

每篇文章包含以下内容：

- **项目简介与核心特性**：功能定位、应用场景、特性一览表
- **硬件架构**：系统组成图、引脚配置、硬件清单
- **固件架构**：文件结构表、模块职责、数据流
- **核心代码深度分析**：从源码仓库提取真实代码，逐行/逐段解析
- **编译与部署**：环境搭建、编译配置、烧录步骤、使用方法
- **项目亮点与适用场景**：技术亮点总结、二次开发方向

## 平台说明

| 分类 | 芯片平台 | 说明 |
|------|---------|------|
| `esp32/` | ESP32, ESP32-S2, ESP32-S3, ESP32-C3/C6, ESP8266 | 乐鑫 ESP 系列，WiFi/BLE 集成 |
| `stm32/` | STM32F1/F4/F7/H7, STM32L 系列 | ST Cortex-M 微控制器 |
| `arduino/` | AVR, ESP32 Arduino Core | Arduino 生态项目 |
| `rp2040/` | RP2040, RP2350 | 树莓派 Pico 系列 |
| `nrf52/` | nRF52832, nRF52840, nRF5340 | Nordic 低功耗 BLE |
| `linux-embedded/` | i.MX, Allwinner, Rockchip | 嵌入式 Linux 平台 |
| `other/` | 其他 MCU | PIC、TI MSP430、RISC-V 等 |

## 关于作者

**蔡浩宇（jun-chy）**

嵌入式开发工程师，专注于 MCU 固件开发、硬件驱动和开源项目研究。

- GitHub: [jun-chy](https://github.com/jun-chy)
- 仓库: [embedded-daily](https://github.com/jun-chy/embedded-daily)

## 许可证

文章内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可证。

引用的开源项目代码版权归原作者所有，具体许可证请参考各项目仓库。
