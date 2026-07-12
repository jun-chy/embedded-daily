# InfiniTime：基于 nRF52 与 FreeRTOS 的开源智能手表固件深度解析

> 日期：2026-07-12  
> 分类：nRF52  
> Star 数：3308  
> 作者：蔡浩宇（jun-chy）

---

## 一、项目链接信息

| 属性 | 内容 |
|------|------|
| 项目名称 | InfiniTime |
| GitHub 地址 | https://github.com/InfiniTimeOrg/InfiniTime |
| 组织 | InfiniTimeOrg |
| Stars | 3308 |
| Forks | 1084 |
| 主要语言 | C 86.7%, C++ 10% |
| 许可证 | GPL-3.0 |
| 创建日期 | 2019-12-05 |
| 最近更新 | 2026-07-12 |
| 最近推送 | 2026-07-05 |
| 最新版本 | 1.16.0 "Starfruit"（2026-01-06） |
| 项目描述 | Firmware for Pinetime smartwatch written in C++ and based on FreeRTOS |
| Topics | bluetooth-low-energy, cpp, embedded, foss, freertos, nrf52, pinetime, smartwatch |

---

## 二、项目简介与核心特性一览

InfiniTime 是一款为 PineTime 智能手表量身打造的开源固件，使用 C++ 编写，运行在 FreeRTOS 实时操作系统之上。PineTime 是一款由 PINE64 社区主导设计的开源硬件智能手表，搭载 Nordic nRF52832 蓝牙 SoC，而 InfiniTime 则是该项目社区中最成熟、生态最完善的固件实现。

InfiniTime 的设计哲学是：在资源极其受限的嵌入式环境（512KB Flash、64KB RAM）中，依然提供完整的智能手表体验，包括表盘 UI、触摸交互、BLE 通信、心率监测、运动传感、通知推送等完整功能，同时保持代码的可读性与可维护性。

### 核心特性一览表

| 特性模块 | 说明 |
|----------|------|
| 显示驱动 | ST7789 LCD 控制器，240x240 分辨率，SPI 8MHz 高速通信 |
| 触摸交互 | CST816S 电容触摸，支持单点、双击、滑动手势识别 |
| 运动传感 | BMA421 三轴加速度计，计步、抬手亮屏、手势唤醒 |
| 心率监测 | HRS3300 光学心率传感器，按需启用以降低功耗 |
| 外部存储 | MX25R6435F 4MB SPI NOR Flash，支持文件系统与资源存储 |
| 蓝牙通信 | 基于 Apache Mynewt NimBLE 协议栈，BLE 5.0 |
| 电源管理 | 深度睡眠、看门狗、DCDC 转换器、电池监测 |
| UI 框架 | 自研 LittleVGL 集成，多表盘切换、应用列表 |
| OTA 升级 | BLE DFU 服务，支持固件无线升级 |
| 配套生态 | Gadgetbridge、Amazfish、ITD 等多平台 companion app |

---

## 三、硬件架构：PineTime 硬件组成

PineTime 是一款以极低成本提供完整智能手表体验的开源硬件平台。其核心是一颗 Nordic nRF52832 蓝牙 SoC，围绕它构建了显示、触摸、传感、存储、电源等完整外设系统。

### 3.1 硬件组成概览

PineTime 的硬件系统可以分为以下几个子系统：

1. **主控子系统**：nRF52832 SoC，集成 Cortex-M4 内核与 BLE 5.0 无线电
2. **显示子系统**：1.3 英寸 ST7789 LCD 面板，240x240 像素，通过 SPI 总线驱动
3. **触摸子系统**：CST816S 电容触摸控制器，通过 I2C（TWI）总线通信
4. **传感子系统**：BMA421 加速度计 + Hrs3300 心率传感器，共享 I2C 总线
5. **存储子系统**：MX25R6435F 4MB SPI NOR Flash，用于存储字体、图片、资源
6. **电源子系统**：170mAh 锂聚合物电池，配备充电电路与电源检测
7. **反馈子系统**：振动马达（用于通知提醒）+ 物理按钮

### 3.2 引脚配置表

InfiniTime 在源码中通过 `Pinetime::PinMap` 命名空间集中管理所有 GPIO 引脚映射。以下是 PineTime 的关键引脚分配：

| 功能模块 | 引脚名称 | GPIO 端口 | 方向 | 说明 |
|----------|----------|-----------|------|------|
| SPI 时钟 | SpiSck | P0.02 | 输出 | SPI 主机时钟信号 |
| SPI MOSI | SpiMosi | P0.03 | 输出 | SPI 主机输出从机输入 |
| SPI MISO | SpiMiso | P0.04 | 输入 | SPI 主机输入从机输出 |
| LCD 片选 | SpiLcdCsn | P0.25 | 输出 | LCD SPI 片选信号 |
| LCD DC | LcdDataCommand | P0.18 | 输出 | LCD 数据/命令选择 |
| LCD 复位 | LcdReset | P0.26 | 输出 | LCD 硬件复位 |
| Flash 片选 | SpiFlashCsn | P0.05 | 输出 | SPI Flash 片选信号 |
| TWI SDA | TwiSda | P0.06 | 双向 | I2C 数据线 |
| TWI SCL | TwiScl | P0.07 | 双向 | I2C 时钟线 |
| 触摸中断 | Cst816sIrq | P0.28 | 输入 | 触摸面板中断信号 |
| 电源检测 | PowerPresent | P0.12 | 输入 | USB 电源接入检测 |
| 按键 | Button | P0.13 | 输入 | 物理按键输入 |
| 振动马达 | Motor | P0.16 | 输出 | 振动马达驱动 |
| 心率中断 | Hrs3300Irq | P0.19 | 输入 | 心率传感器中断（可选） |

### 3.3 硬件清单

| 序号 | 器件名称 | 型号 | 角色 |
|------|----------|------|------|
| 1 | 主控 MCU | Nordic nRF52832 | Cortex-M4 64MHz, 512KB Flash, 64KB RAM, BLE 5.0 |
| 2 | 显示屏 | ST7789 LCD | 1.3" 240x240 IPS 彩色液晶 |
| 3 | 触摸控制器 | CST816S | 电容式多点触摸 |
| 4 | 加速度计 | BMA421 | 三轴加速度，计步与手势 |
| 5 | 心率传感器 | Hrs3300 | 光电式心率监测 |
| 6 | 外部 Flash | MX25R6435F | 4MB SPI NOR Flash |
| 7 | 电池 | 170mAh Li-Po | 锂聚合物可充电电池 |
| 8 | 振动马达 | 贴片马达 | 通知与闹钟反馈 |
| 9 | 按键 | 轻触开关 | 电源/功能按键 |

---

## 四、固件架构：源码目录结构与模块职责

InfiniTime 的源码组织遵循清晰的分层设计理念。驱动层、系统层、应用层各自独立，通过消息队列和回调机制进行通信。

### 4.1 源码目录结构表

| 目录路径 | 职责说明 |
|----------|----------|
| `src/` | 固件核心源码根目录 |
| `src/main.cpp` | 系统入口，硬件初始化与时钟配置 |
| `src/drivers/` | 底层硬件驱动（SPI、TWI、LCD、Flash、传感器） |
| `src/components/` | 功能组件（BLE、显示、电池、设置、闹钟等） |
| `src/components/ble/` | BLE 协议栈封装与 GATT 服务实现 |
| `src/components/ble/services/` | 各类 BLE GATT 服务（音乐、天气、导航、DFU 等） |
| `src/components/ble/services/` | 当前时间、设备信息、电池、心率等服务 |
| `src/systemtask/` | 系统任务，消息循环中枢 |
| `src/displayapp/` | 显示应用框架，UI 渲染与表盘管理 |
| `src/displayapp/screens/` | 各个屏幕界面（表盘、设置、应用） |
| `src/touchhandler/` | 触摸事件处理与手势识别 |
| `src/logging/` | 日志输出（NRF_LOG 封装） |
| `src/libs/` | 第三方库（LittleVGL、date、littlefs 等） |
| `src/FreeRTOS/` | FreeRTOS 配置与端口适配 |
| `src/resources/` | 字体、图片等静态资源 |
| `src/bootloader/` | Bootloader 版本信息与兼容处理 |
| `CMakeLists.txt` | CMake 构建系统入口 |
| `docker/` | Docker 构建环境配置 |
| `doc/` | 项目文档 |

### 4.2 模块职责详解

InfiniTime 的架构可以归纳为以下几个关键层：

**驱动层（Drivers）**：直接操作 nRF52 硬件外设，包括 `SpiMaster`（SPI 主机）、`TwiMaster`（I2C 主机）、`St7789`（LCD 驱动）、`SpiNorFlash`（Flash 驱动）、`Cst816S`（触摸驱动）、`Bma421`（加速度计驱动）、`Hrs3300`（心率驱动）、`Watchdog`（看门狗）。这些驱动不依赖任何上层逻辑，仅提供硬件访问接口。

**系统层（SystemTask）**：作为整个固件的中枢，`SystemTask` 运行在独立的 FreeRTOS 任务中，通过消息队列接收来自中断处理、定时器、其他任务的事件，并分发到对应的处理逻辑。它负责电源管理（睡眠/唤醒）、驱动初始化顺序、应用调度等。

**组件层（Components）**：包含各种功能控制器，如 `BatteryController`（电池状态）、`DateTimeController`（日期时间）、`SettingsController`（用户设置）、`AlarmController`（闹钟）、`MotionController`（运动数据）、`HeartRateController`（心率控制）、`BleController`（蓝牙状态）等。这些控制器管理状态机，但不直接操作硬件。

**BLE 层（Components/Ble）**：基于 NimBLE 协议栈封装，`NimbleController` 统一管理所有 GATT 服务的注册、广播、连接。每个服务（如 `MusicService`、`WeatherService`、`NavigationService`、`DfuService`）独立实现，提供特定的 BLE 功能。

**应用层（DisplayApp）**：管理屏幕显示与用户交互。`DisplayApp` 运行在独立任务中，通过消息队列与 `SystemTask` 通信。各个 `Screen` 子类实现具体界面（表盘、设置菜单、心率监测、音乐控制、导航等），基于 LittleVGL 进行 UI 渲染。

---

## 五、核心代码深度分析

本节选取 InfiniTime 源码中最具代表性的六个片段，逐行解析其设计意图与实现细节。

### 5.1 硬件初始化：main.cpp 驱动对象实例化

以下代码来自 `src/main.cpp`，是固件启动时所有硬件驱动对象的静态实例化过程。这段代码定义了 PineTime 所有外设的通信地址与硬件参数。

```cpp
static constexpr uint8_t touchPanelTwiAddress = 0x15;
static constexpr uint8_t motionSensorTwiAddress = 0x18;
static constexpr uint8_t heartRateSensorTwiAddress = 0x44;

Pinetime::Drivers::SpiMaster spi {Pinetime::Drivers::SpiMaster::SpiModule::SPI0,
                                  {Pinetime::Drivers::SpiMaster::BitOrder::Msb_Lsb,
                                   Pinetime::Drivers::SpiMaster::Modes::Mode3,
                                   Pinetime::Drivers::SpiMaster::Frequencies::Freq8Mhz,
                                   Pinetime::PinMap::SpiSck,
                                   Pinetime::PinMap::SpiMosi,
                                   Pinetime::PinMap::SpiMiso}};

Pinetime::Drivers::Spi lcdSpi {spi, Pinetime::PinMap::SpiLcdCsn};
Pinetime::Drivers::St7789 lcd {lcdSpi, Pinetime::PinMap::LcdDataCommand, Pinetime::PinMap::LcdReset};

Pinetime::Drivers::Spi flashSpi {spi, Pinetime::PinMap::SpiFlashCsn};
Pinetime::Drivers::SpiNorFlash spiNorFlash {flashSpi};

static constexpr uint32_t MaxTwiFrequencyWithoutHardwareBug {0x06200000};
Pinetime::Drivers::TwiMaster twiMaster {NRF_TWIM1, MaxTwiFrequencyWithoutHardwareBug, Pinetime::PinMap::TwiSda, Pinetime::PinMap::TwiScl};
Pinetime::Drivers::Cst816S touchPanel {twiMaster, touchPanelTwiAddress};
Pinetime::Drivers::Bma421 motionSensor {twiMaster, motionSensorTwiAddress};
Pinetime::Drivers::Hrs3300 heartRateSensor {twiMaster, heartRateSensorTwiAddress};
```

**逐行解析：**

```cpp
static constexpr uint8_t touchPanelTwiAddress = 0x15;
```
定义 CST816S 触摸面板的 I2C 从机地址 `0x15`。使用 `constexpr` 确保该值在编译期确定，可以直接用于模板参数或常量折叠优化，避免运行时开销。

```cpp
static constexpr uint8_t motionSensorTwiAddress = 0x18;
```
定义 BMA421 加速度计的 I2C 地址 `0x18`。这是 Bosch BMA421 数据手册中规定的标准从机地址。

```cpp
static constexpr uint8_t heartRateSensorTwiAddress = 0x44;
```
定义 Hrs3300 心率传感器的 I2C 地址 `0x44`。

```cpp
Pinetime::Drivers::SpiMaster spi {Pinetime::Drivers::SpiMaster::SpiModule::SPI0,
```
创建 SPI 主机驱动对象，指定使用 nRF52 的 `SPI0` 硬件模块。nRF52832 有多个 SPI 外设，这里选择 SPI0 用于连接 LCD 和 Flash。

```cpp
                                  {Pinetime::Drivers::SpiMaster::BitOrder::Msb_Lsb,
```
配置 SPI 位序为 MSB 优先（高位先发）。ST7789 和 MX25R6435F 都要求 MSB 优先传输。

```cpp
                                   Pinetime::Drivers::SpiMaster::Modes::Mode3,
```
配置 SPI 工作模式为 Mode3（CPOL=1, CPHA=1）。时钟空闲高电平，第二个跳变沿采样。这是 ST7789 LCD 要求的 SPI 模式。

```cpp
                                   Pinetime::Drivers::SpiMaster::Frequencies::Freq8Mhz,
```
配置 SPI 时钟频率为 8MHz。这是 nRF52 SPI 外设能稳定支持的最高频率，用于保证 LCD 刷新速率。

```cpp
                                   Pinetime::PinMap::SpiSck,
                                   Pinetime::PinMap::SpiMosi,
                                   Pinetime::PinMap::SpiMiso}};
```
通过 `PinMap` 指定 SCK、MOSI、MISO 三个引脚。`PinMap` 将所有引脚定义集中管理，便于硬件版本适配。

```cpp
Pinetime::Drivers::Spi lcdSpi {spi, Pinetime::PinMap::SpiLcdCsn};
```
基于共享的 `spi` 主机对象创建 LCD 专用的 SPI 从设备实例，绑定 LCD 片选引脚 `SpiLcdCsn`。多个 SPI 设备共享同一总线，通过不同的 CS 引脚区分。

```cpp
Pinetime::Drivers::St7789 lcd {lcdSpi, Pinetime::PinMap::LcdDataCommand, Pinetime::PinMap::LcdReset};
```
创建 ST7789 LCD 驱动对象，传入 SPI 从设备句柄、DC（数据/命令选择）引脚和复位引脚。DC 引脚用于区分 SPI 总线上传输的是显示数据还是命令字节。

```cpp
Pinetime::Drivers::Spi flashSpi {spi, Pinetime::PinMap::SpiFlashCsn};
```
同样基于共享 `spi` 创建 Flash 专用 SPI 从设备，绑定 Flash 片选引脚 `SpiFlashCsn`。

```cpp
Pinetime::Drivers::SpiNorFlash spiNorFlash {flashSpi};
```
创建 SPI NOR Flash 驱动对象，用于管理 4MB 外部存储。

```cpp
static constexpr uint32_t MaxTwiFrequencyWithoutHardwareBug {0x06200000};
```
定义 TWI（I2C）最大安全频率。这个特定值 `0x06200000`（约 104MHz 的寄存器配置值）是为了规避 nRF52 TWI 外设的一个已知硬件 Bug。在特定频率下 TWI 可能锁死，此值经过验证可以稳定工作。

```cpp
Pinetime::Drivers::TwiMaster twiMaster {NRF_TWIM1, MaxTwiFrequencyWithoutHardwareBug, Pinetime::PinMap::TwiSda, Pinetime::PinMap::TwiScl};
```
创建 TWI 主机驱动对象，使用 nRF52 的 `TWIM1`（TWI Master 1）外设，三个传感器共享同一条 I2C 总线。

```cpp
Pinetime::Drivers::Cst816S touchPanel {twiMaster, touchPanelTwiAddress};
Pinetime::Drivers::Bma421 motionSensor {twiMaster, motionSensorTwiAddress};
Pinetime::Drivers::Hrs3300 heartRateSensor {twiMaster, heartRateSensorTwiAddress};
```
分别创建触摸面板、加速度计、心率传感器驱动对象，每个对象绑定共享的 `twiMaster` 和各自的 I2C 地址。这种设计允许多个传感器复用同一总线，驱动层通过地址自动区分设备。

### 5.2 系统入口：main.cpp main() 函数

以下代码来自 `src/main.cpp` 的 `main()` 函数，是整个固件启动流程的核心。它完成了时钟初始化、I2C 总线恢复、定时器创建、非初始化数据区处理，最终启动系统任务和 FreeRTOS 调度器。

```cpp
int main() {
  enable_dcdc_regulator();
  logger.Init();

  nrf_drv_clock_init();
  nrf_drv_clock_lfclk_request(nullptr);

  nrfx_clock_lfclk_start();
  while (!nrf_clock_lf_is_running()) {
  }

#if (CLOCK_CONFIG_LF_SRC == NRF_CLOCK_LFCLK_RC)
  nrf_drv_clock_calibration_start(0, calibrate_lf_clock_rc);
#endif

  // Unblock i2c?
  nrf_gpio_cfg(Pinetime::PinMap::TwiScl,
               NRF_GPIO_PIN_DIR_OUTPUT,
               NRF_GPIO_PIN_INPUT_DISCONNECT,
               NRF_GPIO_PIN_NOPULL,
               NRF_GPIO_PIN_S0D1,
               NRF_GPIO_PIN_NOSENSE);
  nrf_gpio_pin_set(Pinetime::PinMap::TwiScl);
  for (uint8_t i = 0; i < 16; i++) {
    nrf_gpio_pin_toggle(Pinetime::PinMap::TwiScl);
    nrf_delay_us(5);
  }
  nrf_gpio_cfg_default(Pinetime::PinMap::TwiScl);

  debounceTimer = xTimerCreate("debounceTimer", 10, pdFALSE, nullptr, DebounceTimerCallback);
  debounceChargeTimer = xTimerCreate("debounceTimerCharge", 200, pdFALSE, nullptr, DebounceTimerChargeCallback);

  Pinetime::BootloaderVersion::SetVersion(NRF_TIMER2->CC[0]);

  if (NoInit_MagicWord == NoInit_MagicValue) {
    dateTimeController.SetCurrentTime(NoInit_BackUpTime);
  } else {
    memset(&__start_noinit_data, 0, (uintptr_t) &__stop_noinit_data - (uintptr_t) &__start_noinit_data);
    NoInit_MagicWord = NoInit_MagicValue;
  }

  systemTask.Start();
  nimble_port_init();
  vTaskStartScheduler();

  for (;;) {
    APP_ERROR_HANDLER(NRF_ERROR_FORBIDDEN);
  }
}
```

**逐段解析：**

```cpp
  enable_dcdc_regulator();
  logger.Init();
```
第一行调用 `enable_dcdc_regulator()` 启用 nRF52 的内部 DC-DC 转换器。与默认的 LDO 模式相比，DC-DC 模式可以显著降低功耗，对于电池供电的智能手表至关重要。第二行初始化日志系统，用于开发调试阶段输出诊断信息。

```cpp
  nrf_drv_clock_init();
  nrf_drv_clock_lfclk_request(nullptr);
```
初始化 nRF52 时钟驱动并请求低频时钟（LFCLK，32.768kHz）。LFCLK 是 BLE 协议栈运行的必要条件，用于维持蓝牙连接的精确定时。`nullptr` 参数表示不使用回调函数，采用同步方式请求。

```cpp
  nrfx_clock_lfclk_start();
  while (!nrf_clock_lf_is_running()) {
  }
```
启动 LFCLK 并阻塞等待直到时钟稳定运行。`while` 空循环是一种忙等待（busy-wait）模式，确保后续依赖 LFCLK 的操作不会因为时钟未就绪而失败。此处等待时间极短（通常数毫秒），不会造成可感知的延迟。

```cpp
#if (CLOCK_CONFIG_LF_SRC == NRF_CLOCK_LFCLK_RC)
  nrf_drv_clock_calibration_start(0, calibrate_lf_clock_rc);
#endif
```
条件编译：如果 LFCLK 来源配置为内部 RC 振荡器（而非外部晶振），则需要定期校准。RC 振荡器精度较低且随温度漂移，Nordic SDK 提供自动校准机制。`calibrate_lf_clock_rc` 是校准完成后的回调函数，用于重新启动下一次校准周期。

```cpp
  // Unblock i2c?
  nrf_gpio_cfg(Pinetime::PinMap::TwiScl,
               NRF_GPIO_PIN_DIR_OUTPUT,
               NRF_GPIO_PIN_INPUT_DISCONNECT,
               NRF_GPIO_PIN_NOPULL,
               NRF_GPIO_PIN_S0D1,
               NRF_GPIO_PIN_NOSENSE);
```
这是一段关键的 I2C 总线恢复逻辑。当系统异常重启（如看门狗超时、固件崩溃）时，如果上一次传输中途失败，从设备可能拉低 SDA 导致总线锁死。这里将 SCL 手动配置为输出模式（开漏，`S0D1` 表示低电平输出0、高电平高阻态），准备手动产生时钟脉冲。

```cpp
  nrf_gpio_pin_set(Pinetime::PinMap::TwiScl);
  for (uint8_t i = 0; i < 16; i++) {
    nrf_gpio_pin_toggle(Pinetime::PinMap::TwiScl);
    nrf_delay_us(5);
  }
```
将 SCL 拉高，然后产生 16 个时钟脉冲（每次翻转间隔 5 微秒）。这 16 个时钟周期足以让任何挂起的 I2C 从设备完成当前字节传输并释放 SDA，从而解除总线锁死。这是 I2C 协议的标准总线恢复手法。

```cpp
  nrf_gpio_cfg_default(Pinetime::PinMap::TwiScl);
```
将 SCL 引脚恢复为默认配置，交还给 TWI 外设控制。

```cpp
  debounceTimer = xTimerCreate("debounceTimer", 10, pdFALSE, nullptr, DebounceTimerCallback);
  debounceChargeTimer = xTimerCreate("debounceTimerCharge", 200, pdFALSE, nullptr, DebounceTimerChargeCallback);
```
创建两个 FreeRTOS 软件定时器用于按键消抖。`debounceTimer` 周期 10ms，用于物理按键消抖；`debounceChargeTimer` 周期 200ms，用于充电状态变化的消抖（充电电路的电源检测信号可能有较长的过渡期）。`pdFALSE` 表示单次触发定时器（非周期性）。

```cpp
  Pinetime::BootloaderVersion::SetVersion(NRF_TIMER2->CC[0]);
```
从 nRF52 的 TIMER2 比较寄存器 `CC[0]` 读取 Bootloader 预先写入的版本号。InfiniTime 的 Bootloader 在跳转到固件前，会将自身版本号写入 TIMER2 的捕获/比较寄存器，固件启动时读取该值以获知当前 Bootloader 版本，用于判断功能兼容性。

```cpp
  if (NoInit_MagicWord == NoInit_MagicValue) {
    dateTimeController.SetCurrentTime(NoInit_BackUpTime);
  } else {
    memset(&__start_noinit_data, 0, (uintptr_t) &__stop_noinit_data - (uintptr_t) &__start_noinit_data);
    NoInit_MagicWord = NoInit_MagicValue;
  }
```
这是一个巧妙的"非初始化数据区"（NoInit）机制。`__start_noinit_data` 和 `__stop_noinit_data` 是链接脚本定义的符号，标记一段在系统复位时不被清零的内存区域。InfiniTime 在此区域保存当前时间（`NoInit_BackUpTime`）和一个魔术字（`NoInit_MagicWord`）。如果魔术字匹配，说明是软复位（如看门狗触发），直接恢复之前保存的时间；否则是冷启动，清零该区域并写入魔术字。这样即使系统意外重启，也能保留大致的时间信息，无需等待 BLE 同步。

```cpp
  systemTask.Start();
  nimble_port_init();
  vTaskStartScheduler();
```
启动 `SystemTask`（创建系统任务并开始执行初始化），初始化 NimBLE 协议栈端口层，最后启动 FreeRTOS 调度器。`vTaskStartScheduler()` 正常情况下不会返回，因为它会切换到第一个任务执行。

```cpp
  for (;;) {
    APP_ERROR_HANDLER(NRF_ERROR_FORBIDDEN);
  }
}
```
如果调度器意外返回（正常不会发生），进入死循环并触发错误处理器。`APP_ERROR_HANDLER` 是 Nordic SDK 的错误处理宏，会记录错误码并触发断言。这是一种防御性编程，确保系统不会在调度器停止后继续执行未定义行为。

### 5.3 BLE 初始化：NimbleController::Init()

以下代码来自 `src/components/ble/NimbleController.cpp`，展示 BLE 协议栈的完整初始化流程，包括服务注册、地址获取、广播启动。

```cpp
void NimbleController::Init() {
  while (!ble_hs_synced()) {
    vTaskDelay(10);
  }

  nptr = this;
  ble_hs_cfg.reset_cb = nimble_on_reset;
  ble_hs_cfg.sync_cb = nimble_on_sync;
  ble_hs_cfg.store_status_cb = ble_store_util_status_rr;

  ble_svc_gap_init();
  ble_svc_gatt_init();

  deviceInformationService.Init();
  currentTimeClient.Init();
  currentTimeService.Init();
  musicService.Init();
  weatherService.Init();
  navService.Init();
  anService.Init();
  dfuService.Init();
  batteryInformationService.Init();
  immediateAlertService.Init();
  heartRateService.Init();
  motionService.Init();
  fsService.Init();

  int rc;
  rc = ble_hs_util_ensure_addr(0);
  ASSERT(rc == 0);
  rc = ble_hs_id_infer_auto(0, &addrType);
  ASSERT(rc == 0);
  rc = ble_svc_gap_device_name_set(deviceName);
  ASSERT(rc == 0);
  rc = ble_svc_gap_device_appearance_set(0xC2);
  ASSERT(rc == 0);

  bleController.Address(std::move(address));

  rc = ble_gatts_start();
  ASSERT(rc == 0);

  RestoreBond();
  StartAdvertising();
}
```

**逐段解析：**

```cpp
  while (!ble_hs_synced()) {
    vTaskDelay(10);
  }
```
阻塞等待 NimBLE 主机（Host Stack）与控制器（Controller）同步完成。BLE 协议栈分为 Host 和 Controller 两层，初始化时需要通过 HCI 完成同步。`vTaskDelay(10)` 让出 CPU 10 个 tick，避免忙等待浪费功耗。

```cpp
  nptr = this;
```
将当前对象指针保存到静态变量 `nptr`。NimBLE 的回调函数是 C 风格的函数指针，无法直接绑定到 C++ 成员函数，因此需要一个静态桥接指针。回调函数内部通过 `nptr` 调用实际的成员方法。

```cpp
  ble_hs_cfg.reset_cb = nimble_on_reset;
  ble_hs_cfg.sync_cb = nimble_on_sync;
  ble_hs_cfg.store_status_cb = ble_store_util_status_rr;
```
配置 NimBLE Host 的回调函数：`reset_cb` 在协议栈重置时调用（如严重错误后）；`sync_cb` 在与 Controller 同步完成时调用（用于启动广播）；`store_status_cb` 使用 NimBLE 内置的轮转（round-robin）存储状态管理器，处理配对信息的持久化。

```cpp
  ble_svc_gap_init();
  ble_svc_gatt_init();
```
初始化 GAP（Generic Access Profile）和 GATT（Generic Attribute Profile）基础服务。GAP 服务包含设备名称、外观等信息；GATT 服务包含服务变更特征值，用于通知客户端服务列表发生变化。

```cpp
  deviceInformationService.Init();
  currentTimeClient.Init();
  currentTimeService.Init();
  musicService.Init();
  weatherService.Init();
  navService.Init();
  anService.Init();
  dfuService.Init();
  batteryInformationService.Init();
  immediateAlertService.Init();
  heartRateService.Init();
  motionService.Init();
  fsService.Init();
```
依次初始化所有自定义 GATT 服务。每个 `Init()` 调用会向 NimBLE 注册对应的 Service UUID 及其 Characteristic。各服务职责如下：
- `deviceInformationService`：设备信息（制造商、型号、固件版本、序列号）
- `currentTimeClient`：当前时间客户端（从手机同步时间）
- `currentTimeService`：当前时间服务（允许手机设置手表时间）
- `musicService`：音乐控制（播放/暂停/上下曲、音量、曲目信息）
- `weatherService`：天气信息推送
- `navService`：导航指令推送
- `anService`：简单通知服务（Alert Notification）
- `dfuService`：OTA 固件升级服务
- `batteryInformationService`：电池电量与充电状态
- `immediateAlertService`：即时告警（来电响铃振动）
- `heartRateService`：心率数据广播
- `motionService`：加速度计数据流
- `fsService`：文件系统访问（通过 BLE 读写外部 Flash）

```cpp
  int rc;
  rc = ble_hs_util_ensure_addr(0);
  ASSERT(rc == 0);
```
确保设备拥有一个有效的 BLE 地址。参数 `0` 表示使用默认的随机或公开地址。如果没有配置地址，此函数会自行产生一个随机静态地址。`ASSERT` 确保地址获取成功，否则触发断言终止运行。

```cpp
  rc = ble_hs_id_infer_auto(0, &addrType);
  ASSERT(rc == 0);
```
自动推断地址类型（公开地址或随机地址）。NimBLE 需要知道地址类型才能正确构建广播包和连接请求。结果存入 `addrType` 成员变量，后续广播时使用。

```cpp
  rc = ble_svc_gap_device_name_set(deviceName);
  ASSERT(rc == 0);
```
设置 GAP 设备名称，即其他设备扫描时看到的蓝牙名称。`deviceName` 通常为 "InfiniTime"。

```cpp
  rc = ble_svc_gap_device_appearance_set(0xC2);
  ASSERT(rc == 0);
```
设置设备外观码 `0xC2`（十进制 194），对应 BLE 外观规范中的"手表"类型。客户端（如手机）可以据此显示对应的图标。

```cpp
  bleController.Address(std::move(address));
```
将获取到的 BLE 地址通过 `std::move` 转移给 `BleController` 组件管理。使用移动语义避免不必要的拷贝，`address` 对象在此之后不再使用。

```cpp
  rc = ble_gatts_start();
  ASSERT(rc == 0);
```
启动 GATT 服务，使之前注册的所有服务对连接的客户端可见。此调用之前注册的服务处于"待激活"状态。

```cpp
  RestoreBond();
  StartAdvertising();
}
```
`RestoreBond()` 从 Flash 中恢复之前保存的配对绑定信息（如绑定密钥），这样已配对的设备重连时无需重新配对。`StartAdvertising()` 启动 BLE 广播，使手表可被其他设备发现和连接。

### 5.4 系统消息循环：SystemTask::Work()

以下代码来自 `src/systemtask/SystemTask.cpp`，是 InfiniTime 系统任务的核心循环。它负责驱动初始化、中断配置，以及通过消息队列分发各类系统事件。

```cpp
void SystemTask::Work() {
  BootErrors bootError = BootErrors::None;

  watchdog.Setup(7, Drivers::Watchdog::SleepBehaviour::Run, Drivers::Watchdog::HaltBehaviour::Pause);
  watchdog.Start();

  if (!nrfx_gpiote_is_init()) {
    nrfx_gpiote_init();
  }

  spi.Init();
  spiNorFlash.Init();
  spiNorFlash.Wakeup();
  fs.Init();
  nimbleController.Init();
  twiMaster.Init();
  touchPanel.Init();
  dateTimeController.Register(this);
  batteryController.Register(this);
  motionSensor.SoftReset();
  alarmController.Init(this);

  twiMaster.Sleep();
  twiMaster.Init();
  motionSensor.Init();
  motionController.Init(motionSensor.DeviceType());
  settingsController.Init();

  displayApp.Register(this);
  displayApp.Start(bootError);
  heartRateSensor.Init();
  heartRateSensor.Disable();
  heartRateApp.Start();
  buttonHandler.Init(this);

  // ... interrupt setup and message loop ...
  while (true) {
    Messages msg;
    if (xQueueReceive(systemTasksMsgQueue, &msg, waitTime) == pdTRUE) {
      switch (msg) {
        case Messages::EnableSleeping:
          wakeLocksHeld--;
          break;
        case Messages::DisableSleeping:
          GoToRunning();
          wakeLocksHeld++;
          break;
        case Messages::GoToRunning:
          GoToRunning();
          break;
        case Messages::GoToSleep:
          GoToSleep();
          break;
        case Messages::OnNewTime:
          if (alarmController.IsEnabled()) {
            alarmController.ScheduleAlarm();
          }
          break;
        case Messages::OnNewNotification:
          if (settingsController.GetNotificationStatus() == Pinetime::Controllers::Settings::Notification::On) {
            if (IsSleeping()) {
              GoToRunning();
            }
            displayApp.PushMessage(Pinetime::Applications::Display::Messages::NewNotification);
          }
          break;
        case Messages::OnTouchEvent:
          if (!touchHandler.ProcessTouchInfo(touchPanel.GetTouchInfo())) {
            break;
          }
          if (state == SystemTaskState::Running) {
            displayApp.PushMessage(Pinetime::Applications::Display::Messages::TouchEvent);
          } else {
            auto gesture = touchHandler.GestureGet();
            if (gesture == Pinetime::Applications::TouchEvents::DoubleTap &&
                settingsController.isWakeUpModeOn(Pinetime::Controllers::Settings::WakeUpMode::DoubleTap)) {
              GoToRunning();
            }
          }
          break;
        // ... more message handlers ...
      }
    }
  }
}
```

**逐段解析：**

```cpp
  BootErrors bootError = BootErrors::None;
```
初始化启动错误标志为"无错误"。后续初始化过程中如果检测到硬件故障，会设置对应的错误位，最终传递给显示应用以在屏幕上展示错误信息。

```cpp
  watchdog.Setup(7, Drivers::Watchdog::SleepBehaviour::Run, Drivers::Watchdog::HaltBehaviour::Pause);
  watchdog.Start();
```
配置看门狗定时器：超时时间 7 秒；系统睡眠时看门狗继续运行（`Run`）；系统暂停（如调试halt）时看门狗暂停（`Pause`）。然后启动看门狗。系统任务必须在 7 秒内定期"喂狗"（刷新看门狗），否则触发系统复位。这是嵌入式系统的最后一道防线，防止固件死锁。

```cpp
  if (!nrfx_gpiote_is_init()) {
    nrfx_gpiote_init();
  }
```
检查并初始化 GPIOTE（GPIO Task Event）模块。GPIOTE 允许 GPIO 引脚产生中断事件，是触摸中断、按键中断、充电检测的基础。条件检查避免重复初始化。

```cpp
  spi.Init();
  spiNorFlash.Init();
  spiNorFlash.Wakeup();
  fs.Init();
```
初始化 SPI 总线、Flash 驱动、唤醒 Flash（从低功耗模式），然后初始化文件系统（LittleFS）挂载在 Flash 之上。注意顺序：必须先初始化硬件总线，再初始化设备驱动，最后初始化基于设备的文件系统。

```cpp
  nimbleController.Init();
  twiMaster.Init();
  touchPanel.Init();
  dateTimeController.Register(this);
  batteryController.Register(this);
  motionSensor.SoftReset();
  alarmController.Init(this);
```
初始化 BLE 控制器（即上一节分析的 `NimbleController::Init()`），然后初始化 TWI 总线和触摸面板。`Register(this)` 让控制器注册为系统任务的观察者，以便在状态变化时推送消息。`motionSensor.SoftReset()` 对加速度计执行软复位，确保已知的初始状态。

```cpp
  twiMaster.Sleep();
  twiMaster.Init();
  motionSensor.Init();
  motionController.Init(motionSensor.DeviceType());
  settingsController.Init();
```
这里有一个看似冗余的 `Sleep()` 后立即 `Init()` 序列。这是因为 BMA421 在软复位后可能导致 TWI 总线状态异常，通过让 TWI 进入睡眠再重新初始化来清除潜在的通信错误。随后真正初始化加速度计，并将设备型号传递给 `MotionController`（不同型号的 BMA421 可能有不同的配置参数）。最后初始化用户设置（从 Flash 读取持久化配置）。

```cpp
  displayApp.Register(this);
  displayApp.Start(bootError);
  heartRateSensor.Init();
  heartRateSensor.Disable();
  heartRateApp.Start();
  buttonHandler.Init(this);
```
注册并启动显示应用任务。显示应用运行在独立的 FreeRTOS 任务中，负责所有 UI 渲染。心率传感器初始化后立即禁用（`Disable()`），因为心率监测是按需启用的，持续运行会显著增加功耗。心率应用任务也在此启动，用于后台心率采集。最后初始化按键处理器。

```cpp
  while (true) {
    Messages msg;
    if (xQueueReceive(systemTasksMsgQueue, &msg, waitTime) == pdTRUE) {
```
进入系统主消息循环。`xQueueReceive` 从消息队列中阻塞等待消息，`waitTime` 指定最大等待时间（超时后返回用于执行周期性任务如看门狗喂狗）。`pdTRUE` 表示成功接收到消息。

```cpp
      switch (msg) {
        case Messages::EnableSleeping:
          wakeLocksHeld--;
          break;
```
处理"允许睡眠"消息：递减唤醒锁计数器。当 `wakeLocksHeld` 降为 0 时，系统可以进入低功耗睡眠。这是一种引用计数机制，确保所有需要保持唤醒的模块都释放后才会真正睡眠。

```cpp
        case Messages::DisableSleeping:
          GoToRunning();
          wakeLocksHeld++;
          break;
```
处理"禁止睡眠"消息：先切换到运行状态，再递增唤醒锁。某些模块（如正在进行的 OTA 升级）需要系统保持唤醒，通过持有唤醒锁来阻止睡眠。

```cpp
        case Messages::GoToRunning:
          GoToRunning();
          break;
        case Messages::GoToSleep:
          GoToSleep();
          break;
```
直接的状态切换消息：`GoToRunning()` 唤醒所有外设（LCD、传感器、提升 CPU 频率），`GoToSleep()` 则让外设进入低功耗模式并关闭显示。

```cpp
        case Messages::OnNewTime:
          if (alarmController.IsEnabled()) {
            alarmController.ScheduleAlarm();
          }
          break;
```
时间更新消息：当通过 BLE 从手机同步了新时间后触发。如果闹钟已启用，调用 `ScheduleAlarm()` 基于新时间重新计算闹钟触发时刻。

```cpp
        case Messages::OnNewNotification:
          if (settingsController.GetNotificationStatus() == Pinetime::Controllers::Settings::Notification::On) {
            if (IsSleeping()) {
              GoToRunning();
            }
            displayApp.PushMessage(Pinetime::Applications::Display::Messages::NewNotification);
          }
          break;
```
新通知消息（如来电、短信、应用通知）：首先检查用户是否开启了通知功能；如果当前处于睡眠状态，先唤醒系统（这样用户才能看到通知）；然后向显示应用推送 `NewNotification` 消息，由显示应用负责弹出通知界面并触发振动。

```cpp
        case Messages::OnTouchEvent:
          if (!touchHandler.ProcessTouchInfo(touchPanel.GetTouchInfo())) {
            break;
          }
```
触摸事件消息：从触摸面板获取原始触摸数据，交给 `touchHandler` 处理。`ProcessTouchInfo` 返回 `false` 表示该触摸事件无需进一步处理（如噪声过滤），直接跳出。

```cpp
          if (state == SystemTaskState::Running) {
            displayApp.PushMessage(Pinetime::Applications::Display::Messages::TouchEvent);
          } else {
            auto gesture = touchHandler.GestureGet();
            if (gesture == Pinetime::Applications::TouchEvents::DoubleTap &&
                settingsController.isWakeUpModeOn(Pinetime::Controllers::Settings::WakeUpMode::DoubleTap)) {
              GoToRunning();
            }
          }
          break;
```
如果系统正在运行，将触摸事件转发给显示应用进行 UI 交互处理。如果系统处于睡眠状态，则检查手势：如果是双击手势且用户启用了"双击唤醒"功能，则唤醒系统。这就是 PineTime 的抬手/双击亮屏功能的实现核心。

### 5.5 NimBLE 中断桥接：硬件 ISR 转发

以下代码来自 `src/main.cpp`，展示 InfiniTime 如何将 nRF52 的硬件中断桥接到 NimBLE 协议栈的 FreeRTOS 适配层。

```cpp
static void (*radio_isr_addr)();
static void (*rng_isr_addr)();
static void (*rtc0_isr_addr)();

extern "C" {
void RADIO_IRQHandler(void) {
  ((void (*)()) radio_isr_addr)();
}

void RNG_IRQHandler(void) {
  ((void (*)()) rng_isr_addr)();
}

void RTC0_IRQHandler(void) {
  ((void (*)()) rtc0_isr_addr)();
}

void npl_freertos_hw_set_isr(int irqn, void (*addr)()) {
  switch (irqn) {
    case RADIO_IRQn:
      radio_isr_addr = addr;
      break;
    case RNG_IRQn:
      rng_isr_addr = addr;
      break;
    case RTC0_IRQn:
      rtc0_isr_addr = addr;
      break;
    default:
      break;
  }
}

uint32_t npl_freertos_hw_enter_critical(void) {
  uint32_t ctx = __get_PRIMASK();
  __disable_irq();
  return (ctx & 0x01);
}

void npl_freertos_hw_exit_critical(uint32_t ctx) {
  if (ctx == 0) {
    __enable_irq();
  }
}
```

**逐段解析：**

```cpp
static void (*radio_isr_addr)();
static void (*rng_isr_addr)();
static void (*rtc0_isr_addr)();
```
定义三个静态函数指针变量，用于存储 NimBLE 注册的中断服务程序地址。NimBLE 协议栈需要三个硬件中断：RADIO（射频收发）、RNG（随机数生成器，用于加密）、RTC0（实时时钟，用于 BLE 调度定时）。初始值为 `nullptr`。

```cpp
extern "C" {
```
使用 `extern "C"` 包裹以下函数，禁止 C++ 名称修饰（name mangling）。因为这些函数需要被链接器作为中断向量表入口和 NimBLE C 库的回调，必须保持 C 链接约定。

```cpp
void RADIO_IRQHandler(void) {
  ((void (*)()) radio_isr_addr)();
}
```
这是 nRF52 RADIO 外设的硬件中断服务程序入口。中断向量表中 `RADIO_IRQHandler` 的地址在硬件触发 RADIO 中断时被自动调用。函数内部将之前注册的 `radio_isr_addr` 转换为函数指针并调用，从而将控制权转交给 NimBLE 的射频中断处理逻辑。

```cpp
void RNG_IRQHandler(void) {
  ((void (*)()) rng_isr_addr)();
}

void RTC0_IRQHandler(void) {
  ((void (*)()) rtc0_isr_addr)();
}
```
同理，RNG 和 RTC0 的中断也通过函数指针转发。这种间接调用的设计是因为中断向量表在编译时固定，而 NimBLE 的实际处理函数在运行时动态注册，需要这层桥接。

```cpp
void npl_freertos_hw_set_isr(int irqn, void (*addr)()) {
```
这是 NimBLE 的 NPL（NimBLE Porting Layer）接口函数，用于在运行时注册中断处理函数。NimBLE 在初始化时调用此函数，将自己内部的处理函数地址传递给固件。

```cpp
  switch (irqn) {
    case RADIO_IRQn:
      radio_isr_addr = addr;
      break;
    case RNG_IRQn:
      rng_isr_addr = addr;
      break;
    case RTC0_IRQn:
      rtc0_isr_addr = addr;
      break;
    default:
      break;
  }
}
```
根据中断号将处理函数地址保存到对应的静态变量。当硬件中断触发时，`*IRQHandler` 会调用此处注册的函数。`default` 分支忽略未识别的中断号。

```cpp
uint32_t npl_freertos_hw_enter_critical(void) {
  uint32_t ctx = __get_PRIMASK();
  __disable_irq();
  return (ctx & 0x01);
}
```
进入临界区函数。首先通过 `__get_PRIMASK()` 读取当前中断使能状态（PRIMASK 寄存器 bit0 为 1 表示全局中断禁用），然后调用 `__disable_irq()` 禁用所有中断。返回值 `ctx & 0x01` 保存了进入前的中断状态，用于后续恢复。这种"保存-禁用"模式支持嵌套调用：如果中断已经被禁用，不会错误地重新启用。

```cpp
void npl_freertos_hw_exit_critical(uint32_t ctx) {
  if (ctx == 0) {
    __enable_irq();
  }
}
}
```
退出临界区函数。只有当进入前中断是启用的（`ctx == 0`）才重新启用中断。如果进入前中断已被禁用（嵌套场景），保持禁用状态，由最外层的调用负责最终启用。这是嵌入式实时系统临界区保护的标准模式。

### 5.6 GPIOTE 事件处理：中断到消息的转换

以下代码来自 `src/main.cpp`，展示 GPIO 中断如何被转换为系统消息，是 InfiniTime 事件驱动架构的典型实现。

```cpp
void nrfx_gpiote_evt_handler(nrfx_gpiote_pin_t pin, nrf_gpiote_polarity_t action) {
  if (pin == Pinetime::PinMap::Cst816sIrq) {
    systemTask.PushMessage(Pinetime::System::Messages::OnTouchEvent);
    return;
  }

  BaseType_t xHigherPriorityTaskWoken = pdFALSE;

  if (pin == Pinetime::PinMap::PowerPresent and action == NRF_GPIOTE_POLARITY_TOGGLE) {
    xTimerStartFromISR(debounceChargeTimer, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
  } else if (pin == Pinetime::PinMap::Button) {
    xTimerStartFromISR(debounceTimer, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
  }
}
```

**逐段解析：**

```cpp
void nrfx_gpiote_evt_handler(nrfx_gpiote_pin_t pin, nrf_gpiote_polarity_t action) {
```
这是 GPIOTE 事件的统一回调函数。当任何已配置的 GPIO 引脚触发中断时，nRFx 驱动会调用此函数，传入触发引脚号和触发极性（上升沿、下降沿或跳变）。此函数运行在中断上下文（ISR）中，必须尽量简短。

```cpp
  if (pin == Pinetime::PinMap::Cst816sIrq) {
    systemTask.PushMessage(Pinetime::System::Messages::OnTouchEvent);
    return;
  }
```
触摸中断处理：CST816S 触摸面板在有触摸事件时拉低中断引脚。此处直接向系统任务推送 `OnTouchEvent` 消息并返回。触摸处理不需要消抖（CST816S 内部已做处理），因此直接转发。`PushMessage` 内部使用 FreeRTOS 队列的 ISR 安全版本。

```cpp
  BaseType_t xHigherPriorityTaskWoken = pdFALSE;
```
声明一个标志变量，用于通知 FreeRTOS 是否有更高优先级的任务被唤醒。这是 FreeRTOS ISR 编程的标准模式：ISR 中不能直接操作任务上下文，但可以标记需要任务切换，在退出 ISR 时由内核执行切换。

```cpp
  if (pin == Pinetime::PinMap::PowerPresent and action == NRF_GPIOTE_POLARITY_TOGGLE) {
```
电源接入检测：当 `PowerPresent` 引脚发生电平跳变（插入或拔出 USB 电源）时触发。使用 `and`（C++ 关键字，等价于 `&&`）同时检查引脚和触发极性，确保只处理跳变事件。

```cpp
    xTimerStartFromISR(debounceChargeTimer, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
```
从 ISR 启动充电消抖定时器（200ms 后触发回调）。不直接处理充电状态变化是因为电源检测信号可能存在抖动（如接触不良）。`xTimerStartFromISR` 是 FreeRTOS 定时器的 ISR 安全版本。`portYIELD_FROM_ISR` 检查标志位，如果有高优先级任务被唤醒，则在 ISR 退出时立即进行任务切换。

```cpp
  } else if (pin == Pinetime::PinMap::Button) {
    xTimerStartFromISR(debounceTimer, &xHigherPriorityTaskWoken);
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
  }
}
```
按键中断处理：物理按键按下/释放时触发，启动 10ms 消抖定时器。消抖定时器在 10ms 后回调 `DebounceTimerCallback`，此时按键电平已稳定，可以可靠地判断按下或释放状态。同样使用 `portYIELD_FROM_ISR` 确保及时的任务调度。

这段代码体现了 InfiniTime 中断处理的核心设计原则：**ISR 中只做最少的工作**（推送消息或启动定时器），将实际业务逻辑推迟到任务上下文中执行。这保证了中断响应的实时性，同时避免了在中断中执行耗时操作导致的中断延迟。

---

## 六、API 使用指南

### 6.1 BLE GATT 服务列表

InfiniTime 暴露了丰富的 BLE GATT 服务，以下为主要服务及其 UUID：

| 服务名称 | UUID | 说明 |
|----------|------|------|
| Device Information | `0000180A-0000-1000-8000-00805F9B34FB` | 设备制造商、型号、固件版本、硬件版本、序列号 |
| Battery Service | `0000180F-0000-1000-8000-00805F9B34FB` | 电池电量百分比、充电状态 |
| Current Time Service | `00001805-0000-1000-8000-00805F9B34FB` | 当前时间同步 |
| Immediate Alert | `00001802-0000-1000-8000-00805F9B34FB` | 来电告警（振动马达） |
| Heart Rate Service | `0000180D-0000-1000-8000-00805F9B34FB` | 心率测量值广播 |
| Music Service | `00000000-78FC-48FE-8E23-433B3A1942D0` | 音乐播放控制与状态 |
| Weather Service | `00000001-78FC-48FE-8E23-433B3A1942D0` | 天气数据推送 |
| Navigation Service | `00000002-78FC-48FE-8E23-433B3A1942D0` | 导航指令推送 |
| Alert Notification | `00001811-0000-1000-8000-00805F9B34FB` | 通知消息推送 |
| DFU Service | `00001530-1212-EFDE-1523-785FEABCD123` | OTA 固件升级 |
| FS Service | `00001534-1212-EFDE-1523-785FEABCD123` | 文件系统读写 |

### 6.2 配套应用

InfiniTime 拥有丰富的配套应用生态，支持多种平台：

| 应用名称 | 平台 | 说明 |
|----------|------|------|
| Gadgetbridge | Android | 开源综合管理应用，支持通知同步、音乐控制、天气推送、固件升级 |
| Amazfish | Sailfish OS / Linux | Linux 系开源手表管理工具，支持多种手表协议 |
| ITD (InfiniTime Daemon) | Linux | 命令行守护进程，可脚本化控制手表功能 |
| Siglo | Linux | GTK 图形界面管理工具 |
| WatchMate | Linux/Android | Rust 实现的跨平台管理工具 |

### 6.3 cURL 示例：通过 BLE 与 InfiniTime 交互

以下示例使用 `bluetoothctl` 或 `gatttool` 与 InfiniTime 进行 BLE 交互（以 Linux `bluetoothctl` 为例）：

```bash
# 扫描设备
bluetoothctl scan on

# 连接手表（替换为实际 MAC 地址）
bluetoothctl connect AA:BB:CC:DD:EE:FF

# 读取电池电量（Battery Service UUID: 0x2A19）
bluetoothctl menu gatt
bluetoothctl list-attributes AA:BB:CC:DD:EE:FF
bluetoothctl select-attribute 00002A19-0000-1000-8000-00805F9B34FB
bluetoothctl read

# 写入当前时间（Current Time Service，UUID: 0x2A2B）
# 时间数据格式：年(2字节LE) 月 日 时 分 秒 星期
bluetoothctl select-attribute 00002A2B-0000-1000-8000-00805F9B34FB
bluetoothctl write "0xEA 0x07 0x07 0x0C 0x0E 0x1E 0x00 0x02"

# 触发即时告警（振动马达）
bluetoothctl select-attribute 00002A06-0000-1000-8000-00805F9B34FB
bluetoothctl write "0x01"
```

使用 ITD 命令行工具的示例：

```bash
# 安装 ITD
pip install itd

# 同步时间
itctl time sync

# 推送通知
itctl notify -t "标题" -b "这是通知正文内容"

# 查看电池信息
itctl battery

# 上传固件进行 OTA 升级
itctl firmware update firmware.zip

# 设置心率监测间隔
itctl heartrate set 30
```

---

## 七、编译与部署

### 7.1 Docker 构建（推荐）

InfiniTime 提供了预配置的 Docker 构建环境，无需手动安装工具链：

```bash
# 克隆仓库
git clone https://github.com/InfiniTimeOrg/InfiniTime.git
cd InfiniTime
git submodule update --init --recursive

# 使用 Docker 构建固件
docker run --rm -v $(pwd):/work infinitime/infinitime-build

# 指定构建目标
docker run --rm -v $(pwd):/work infinitime/infinitime-build \
  cmake --build build -j$(nproc)

# 构建特定变体（如带资源压缩）
docker run --rm -v $(pwd):/work infinitime/infinitime-build \
  cmake -B build -S . -DINFINTIME_BUILD_TYPE=Release
```

构建成功后，输出文件位于 `build/output/` 目录：
- `pinetime-mcuboot-app.img`：MCUBoot 兼容的固件镜像（用于 OTA 升级）
- `pinetime-mcuboot-app.hex`：完整 Hex 文件（用于 SWD 烧录）
- `dfu.zip`：BLE DFU 升级包

### 7.2 CMake 手动构建

如果不使用 Docker，需要手动安装以下依赖：
- ARM GCC 工具链（`arm-none-eabi-gcc`）
- CMake 3.10+
- Ninja 或 Make
- nRF5 SDK（通过子模块自动获取）

```bash
# 安装工具链（Ubuntu/Debian）
sudo apt install gcc-arm-none-eabi cmake ninja-build

# 配置构建
cmake -B build -S . -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DUSE_OPENOCD=1

# 编译
cmake --build build -j$(nproc)

# 查看构建产物
ls build/output/
```

### 7.3 SWD 烧录

PineTime 通过 SWD（Serial Wire Debug）接口烧录固件，需要外部编程器（如 ST-Link、J-Link、DAPLink、Raspberry Pi GPIO）。

使用 OpenOCD 烧录：

```bash
# 使用 ST-Link 烧录完整镜像
openocd -f interface/stlink.cfg \
  -f target/nrf52.cfg \
  -c "init; reset init; program build/output/pinetime-mcuboot-app.hex verify; reset; exit"

# 仅烧录 Bootloader
openocd -f interface/stlink.cfg \
  -f target/nrf52.cfg \
  -c "init; reset init; program bootloader/bootloader-*.hex verify; reset; exit"

# 擦除整个 Flash
openocd -f interface/stlink.cfg \
  -f target/nrf52.cfg \
  -c "init; reset init; nrf5 mass_erase; exit"
```

使用 J-Link 烧录：

```bash
# 通过 JLink Commander 烧录
JLinkExe -device nrf52 -if swd -speed 4000 -autoconnect 1
> loadfile build/output/pinetime-mcuboot-app.hex
> r
> g
> exit
```

使用 SWD 烧录后首次启动，手表会进入 InfiniTime 的初始设置流程。若已安装 Bootloader（MCUBoot），后续可通过 BLE OTA 进行无线升级。

---

## 八、项目亮点与适用场景

### 8.1 项目亮点

1. **极致的资源管理**：在仅 64KB RAM、512KB Flash 的约束下，实现了完整的智能手表功能。通过 NoInit 内存区保存时间、按需启用心率传感器、深度睡眠等手段，将功耗控制到极致。

2. **优雅的事件驱动架构**：`SystemTask` 消息循环将所有硬件中断、定时器事件、BLE 事件统一编排，模块间通过消息队列解耦。中断处理只做最小工作（推送消息），保证实时响应。

3. **完整的 BLE 生态**：实现了十余个 GATT 服务，覆盖通知、音乐、天气、导航、心率、OTA 等场景，配合 Gadgetbridge、ITD 等开源配套应用，形成完整的开源智能手表生态。

4. **NimBLE 协议栈适配**：相比常见的 BlueZ 或 Nordic SoftDevice，NimBLE 是专为嵌入式设计的轻量级 BLE 协议栈。InfiniTime 通过 NPL（NimBLE Porting Layer）实现了与 FreeRTOS 的深度集成，包括中断桥接、临界区管理。

5. **I2C 总线恢复机制**：针对 nRF52 TWI 硬件 Bug 和总线锁死场景，实现了软件时钟脉冲恢复逻辑，大幅提升系统在异常重启后的恢复能力。

6. **OTA 固件升级**：通过 MCUBoot Bootloader 和 BLE DFU 服务，用户可以完全无线升级固件，无需任何硬件工具。

### 8.2 适用场景

- **嵌入式学习与研究**：InfiniTime 是学习 RTOS、BLE 协议栈、低功耗设计、嵌入式 C++ 的优秀实战项目，代码结构清晰、注释完善。
- **可穿戴设备开发参考**：对于开发自定义智能手表、健康监测手环的团队，InfiniTime 的驱动层、电源管理、传感器集成方案可直接借鉴。
- **nRF52 平台开发模板**：项目展示了 nRF52 的 SPI、TWI、GPIOTE、RTC、Radio 等外设的完整用法，是 nRF52 开发的参考工程。
- **BLE 外设开发参考**：NimBLE 的集成方式、GATT 服务设计模式、配对绑定管理，对任何 BLE 外设开发都有参考价值。
- **开源硬件社区贡献**：PineTime 作为开源硬件，配合 InfiniTime 开源固件，为爱好者提供了从硬件到软件完全自主可控的智能手表平台。

---

## 九、总结

InfiniTime 是嵌入式开源社区中的一颗明珠。它证明了在 64KB RAM、512KB Flash 的极限资源约束下，依然可以用 C++ 构建出功能完整、架构优雅、体验流畅的智能手表固件。

从代码层面看，InfiniTime 的设计处处体现了嵌入式工程的智慧：NoInit 内存区保存时间跨越重启、I2C 总线软件恢复应对硬件 Bug、中断到消息的转换实现事件驱动、唤醒锁引用计数管理睡眠策略、NimBLE NPL 桥接实现协议栈移植。这些技术细节不仅适用于 PineTime，更是整个嵌入式领域的通用最佳实践。

从生态层面看，InfiniTime 与 Gadgetbridge、ITD、Amazfish 等开源配套应用形成了完整的开源智能手表生态链，用户无需依赖任何闭源软件即可享受完整的智能手表体验。这种"硬件开源 + 固件开源 + 配套开源"的全栈开源模式，在消费电子领域极为罕见。

对于嵌入式开发者而言，深入研读 InfiniTime 源码，尤其是 `main.cpp` 的初始化流程、`SystemTask::Work()` 的消息循环、`NimbleController::Init()` 的 BLE 协议栈集成，将获得关于 RTOS 编程、低功耗设计、BLE 协议栈、嵌入式 C++ 的系统性认知提升。

---

> 作者：蔡浩宇（jun-chy）  
> 日期：2026-07-12  
> 项目地址：https://github.com/InfiniTimeOrg/InfiniTime
