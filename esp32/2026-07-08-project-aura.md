# Project Aura AQ 深度解析：ESP32-S3 全功能空气质量监测站固件架构与源码剖析

> 📅 日期：2026-07-08 ｜ 📂 分类：ESP32 / 空气质量监测 ｜ ⭐ Stars：683 ｜ ✍️ 作者：蔡浩宇（jun-chy）

---

## 项目链接

| 属性 | 信息 |
|------|------|
| GitHub 地址 | [21cncstudio/project_aura](https://github.com/21cncstudio/project_aura) |
| 作者 | Volodymyr Papush（21CNCStudio） |
| 许可证 | GPL-3.0-or-later（商业使用需另行授权） |
| 最近更新 | 2026-07-07 |
| 创建时间 | 2026-01-22 |
| 主要语言 | C（Arduino ESP32 Core 3.1.1 / ESP-IDF 5.3.x） |
| 当前版本 | 1.1.5 |
| 固件大小 | 16MB Flash |

Project Aura AQ 是一个面向"成品级"体验的开源 ESP32-S3 空气质量监测站。它并非又一个临时拼凑的传感器原型，而是一台带有触摸屏 UI、本地 Web 仪表盘、OTA 升级、MQTT / Home Assistant 集成以及可选 0-10V 通风控制的完整设备。该项目于 2026 年 1 月创建，短短半年内即获得 683 stars，是近期 ESP32 生态中增长最快、工程质量最高的空气质量项目之一。

---

## 一、项目简介与核心特性

Project Aura 围绕 Sensirion SEN66 多合一传感器构建，可同时监测颗粒物（PM0.5/1/2.5/4/10）、气体（CO/CO2/VOC/NOx/HCHO 及可选电化学气体）、气候（温湿度/绝对湿度/气压）。它支持 4.3" 和 7" Waveshare ESP32-S3 触摸屏，固件自动探测多种硬件，无需云端即可本地运行。

### 核心特性一览

| 维度 | 特性 |
|------|------|
| 颗粒物 | PM0.5（数量浓度）/ PM1 / PM2.5 / PM4 / PM10 |
| 气体 | CO、CO2、VOC、NOx、HCHO，外加一路可选电化学气体（NH3/SO2/NO2/H2S/O3） |
| 气候 | 温度、湿度、绝对湿度、气压（3h/24h 变化趋势） |
| UI | LVGL 流畅触摸界面、夜间模式、自定义主题、9 种语言、状态指示 |
| Web | 本地仪表盘 `/dashboard`、实时图表、事件日志、设置同步、OTA 升级 |
| 网络 | Wi-Fi AP 配网 + mDNS、支持 WPA2-Enterprise（PEAP/TTLS/TLS） |
| 集成 | MQTT 自动发现、Home Assistant ready、可选 0-10V DAC 通风控制 |
| 可靠性 | Safe Boot 崩溃回滚、看门狗、内存监控、OTA 回滚保护 |
| 硬件 | 自动探测 BMP580/BMP388/BMP390/DPS310 气压、PCF8523/DS3231 RTC |

---

## 二、硬件架构

### 系统组成图

```
                          ┌─────────────────────────────────────┐
                          │      ESP32-S3 (16MB Flash)          │
                          │  ┌───────────┐   ┌──────────────┐   │
                          │  │  Core 0   │   │   Core 1     │   │
                          │  │ 网络/MQTT │   │  LVGL UI     │   │
                          │  │ Web/OTA   │   │  传感器轮询   │   │
                          │  └─────┬─────┘   └──────┬───────┘   │
                          │        │                │           │
                          │   ┌────┴────────────────┴────┐      │
                          │   │   I2C Bus (GPIO8/9)      │      │
                          │   │   100kHz, 共享总线        │      │
                          │   └──┬─────┬─────┬─────┬─────┘     │
                          └──────┼─────┼─────┼─────┼───────────┘
                                 │     │     │     │
                    ┌────────────┘     │     │     └────────────┐
                    ▼                  ▼     ▼                  ▼
              ┌──────────┐    ┌──────────┐ ┌──────────┐  ┌──────────┐
              │  SEN66   │    │ SFA30/40 │ │ SEN0466  │  │ GP8403   │
              │ PM/CO2/  │    │  HCHO    │ │   CO     │  │ 0-10V DAC│
              │ VOC/NOx/ │    │ 甲醛     │ │ 一氧化碳 │  │ 通风控制  │
              │ 温湿度   │    │          │ │          │  │          │
              │ 0x6B     │    │ 0x5D     │ │ 0x74     │  │ 0x58     │
              └──────────┘    └──────────┘ └──────────┘  └────┬─────┘
                                                                   │
                    ┌────────────┐    ┌──────────┐                 │
                    │  BMP580/   │    │ PCF8523/ │                 ▼
                    │  BMP388/   │    │ DS3231   │           ┌──────────┐
                    │  DPS310    │    │  RTC     │           │ 外部风机  │
                    │  气压      │    │  时钟    │           │ / 执行器  │
                    └────────────┘    └──────────┘           └──────────┘

                    ┌──────────────────────────────────────────┐
                    │  Waveshare ESP32-S3-Touch-LCD (4.3"/7")   │
                    │  ├─ RGB LCD (800x480)                     │
                    │  ├─ GT911 电容触摸 (I2C 0x5D)             │
                    │  └─ 背光控制                              │
                    └──────────────────────────────────────────┘
```

### 引脚配置

| 组件 | ESP32-S3 引脚 | 说明 |
|------|--------------|------|
| 3V3 | 3V3 | 外部 I2C 传感器供电 |
| GND | GND | 公共地 |
| I2C SDA | GPIO 8 | 所有传感器和 DAC 共享总线 |
| I2C SCL | GPIO 9 | 所有传感器和 DAC 共享总线 |

> 显示与触摸集成在开发板上，无需额外接线。DAC 模拟输出由 GP8403 模块的 `VOUT0` 产生，而非 ESP32 直连引脚。I2C 总线固定运行在 100kHz，所有外设共用 `I2C_NUM_0` 端口。

### 硬件清单表

| 组件 | 型号 | I2C 地址 | 角色 |
|------|------|----------|------|
| 核心板 | Waveshare ESP32-S3-Touch-LCD-4.3/7.0（16MB） | — | 主控 + 显示 + 触摸 |
| 主传感器 | Sensirion SEN66 | 0x6B | PM/CO2/VOC/NOx/温湿度 |
| 甲醛 | Sensirion SFA30 或 SFA40 | 0x5D | HCHO（自动探测） |
| 一氧化碳 | DFRobot SEN0466 | 0x74 | 电化学 CO（可选） |
| 可选气体槽 | DFRobot SEN0469/0470/0471/0467/0472 | 0x75 | NH3/SO2/NO2/H2S/O3 |
| 气压 | BMP580/BMP388/BMP390/DPS310 | 0x46/0x47/0x77/0x76 | 气压（自动探测） |
| RTC | PCF8523 或 DS3231 | 0x68 | 实时时钟（自动探测） |
| DAC 输出 | DFRobot GP8403（DFR0971） | 0x58 | 0-10V 通风控制（可选） |
| 触摸 | GT911 | 0x5D/0x14 | 电容触摸屏 |

---

## 三、固件架构

Project Aura 采用清晰的分层管理器架构。核心模块位于 `src/core/`，功能管理器位于 `src/modules/`，UI 位于 `src/ui/`，Web 页面位于 `src/web/`，传感器驱动位于 `src/drivers/`。

### 文件结构表

| 目录 | 职责 | 关键文件 |
|------|------|----------|
| `src/config/` | 全局配置与数据结构 | `AppConfig.h`（所有常量/地址/阈值）、`AppData.h`（SensorData） |
| `src/core/` | 启动、可靠性、运行时状态 | `AppInit.cpp`、`BootPolicy.cpp`、`Watchdog.cpp`、`OtaRollback.cpp`、`NetworkPlane.cpp` |
| `src/drivers/` | I2C 传感器底层驱动 | `Sen66.cpp`、`Sfa30/40.cpp`、`Sen0466.cpp`、`Gp8403.cpp`、`Bmp580.cpp` |
| `src/modules/` | 功能管理器 | `SensorManager.cpp`、`FanControl.cpp`、`MqttManager.cpp`、`NetworkManager.cpp`、`TimeManager.cpp` |
| `src/ui/` | LVGL 界面与本地化 | `UiController.cpp`、`ThemeManager.cpp`、`strings/`（9 语言） |
| `src/web/` | Web 仪表盘与 API | `WebHandlers.cpp`、`WebOtaHandlers.cpp`、`WebChartsApiHandlers.cpp` |
| `test/` | 原生主机测试 | Unity 测试框架，覆盖传感器逻辑、MQTT 负载构建 |

### 模块职责与数据流

固件采用单文件聚合所有管理器实例的设计（`main.cpp`），在 `setup()` 中按序初始化，在 `loop()` 中协调轮询。数据流为：传感器（I2C）→ `SensorManager`（过滤/校验）→ `SensorData`（全局数据结构）→ UI/Web/MQTT 三路消费者。`FanControl` 反向读取 `SensorData` 计算自动通风需求，再通过 I2C 写入 GP8403 DAC。

```
传感器(I2C) ──► SensorManager ──► SensorData ──┬─► UiController (LVGL)
                   │  (过滤/校验/                 ├─► WebRuntimeState (HTTP API)
                   │   合理性检查)                 ├─► MqttManager (MQTT 发布)
                   │                              └─► FanControl ──► GP8403 DAC ──► 风机
                   └─► PressureHistory ──► LittleFS 存储
```

---

## 四、核心代码深度分析

### 4.1 系统入口与主循环（main.cpp）

固件入口 `setup()` 完成从启动状态恢复、I2C 总线修复、管理器初始化、外设初始化到 UI 和网络平面启动的完整流程。这是理解整个系统启动顺序的关键。

```cpp
void setup()
{
    delay(3000);
    Serial.begin(115200);
    Logger::begin(Serial, static_cast<Logger::Level>(Config::LOG_LEVEL));
    Logger::setSerialOutputEnabled(Config::LOG_SERIAL_OUTPUT);
    Logger::setSensorsSerialOutputEnabled(Config::LOG_SERIAL_SENSORS_OUTPUT);
    OtaRollback::logCurrentAppState();
```

**逐段解析**：启动延迟 3 秒是为等待 USB-CDC 串口就绪，避免早期日志丢失。`Logger` 是一个分级日志系统（0=error 到 3=debug），支持独立控制传感器日志输出开关。`OtaRollback::logCurrentAppState()` 在启动第一时间记录 OTA 回滚状态——这是 Safe Boot 机制的核心，用于判断本次启动是否需要标记为有效。

```cpp
    memoryMonitor.begin(Config::MEM_LOG_INTERVAL_MS);
    boot_start_ms = millis();

    StorageManager::BootAction boot_action = AppInit::handleBootState();
    AppInit::recoverI2cBus(static_cast<gpio_num_t>(I2C_SDA_PIN),
                           static_cast<gpio_num_t>(I2C_SCL_PIN));
```

**逐段解析**：`boot_start_ms` 记录启动时刻，用于后续判断是否达到稳定启动阈值（`SAFE_BOOT_STABLE_MS = 60000`，即 60 秒无崩溃才算稳定）。`handleBootState()` 读取上次启动计数和崩溃复位原因，决定本次启动策略（正常/回滚/恢复出厂）。`recoverI2cBus()` 是关键的鲁棒性设计——如果上次崩溃发生在 I2C 传输中途，总线可能被某个从设备拉低 SDA 卡死，此处通过手动时钟脉冲（clock stretching recovery）释放总线。

```cpp
    AppInit::Context init_ctx{
        storage, networkManager, mqttManager, connectivityRuntime,
        mqttRuntimeState, chartsRuntimeState, webRuntimeState, webUiBridge,
        displayThresholds, networkCommandQueue, sensorManager, timeManager,
        themeManager, backlightManager, nightModeManager, fanControl,
        pressureHistory, chartsHistory, uiController, currentData,
        night_mode, temp_units_c, led_indicators_enabled, alert_blink_enabled,
        co2_asc_enabled, temp_offset, hum_offset
    };

    AppInit::initManagersAndConfig(init_ctx, boot_action);
    auto *board = AppInit::initBoardAndPeripherals(init_ctx);
    sdCardManager.begin(board);
    dailyExtremaHistory.begin(sdCardManager, temp_units_c);
    networkManager.attachDailyHistory(sdCardManager, dailyExtremaHistory);
    AppInit::initLvglAndUi(init_ctx, board);
```

**逐段解析**：`AppInit::Context` 是一个聚合了所有管理器引用的上下文结构体，避免在初始化函数间传递大量参数。初始化严格按依赖顺序进行：先初始化管理器和配置（`initManagersAndConfig`），再初始化板级外设（显示/触摸/背光，返回 `board` 指针），然后挂载 SD 卡和每日极值历史，最后初始化 LVGL 和 UI。这个顺序确保 UI 渲染时传感器和存储已就绪。

```cpp
    Watchdog::setup(TASK_WDT_TIMEOUT_MS);
    if (!safe_restart_init()) {
        LOGW("Restart", "Core0 restart task init failed; controlled restart requests will abort");
    }
    webUiBridge.setDispatchMode(WebUiBridge::DispatchMode::DeferredReply);
    network_plane_running = NetworkPlane::start(network_plane_context);
```

**逐段解析**：`Watchdog` 设置 180 秒（`TASK_WDT_TIMEOUT_MS`）任务看门狗超时。`safe_restart_init()` 创建一个专用的 Core 0 重启任务——这是一个精妙的设计：直接在主循环中调用 `esp_restart()` 会使用小的 IPC 任务栈导致 "Cache disabled" panic，改为委托给 Core 0 专用任务避免了竞态。`NetworkPlane::start()` 尝试把网络轮询移到独立任务（Core 0），若失败则回退到主循环内轮询（`network_plane_running = false` 时主循环自己处理网络）。

主循环 `loop()` 是固件的调度核心，处理传感器轮询、网络、UI、OTA 窗口管理和稳定启动判定：

```cpp
    SensorManager::PollResult sensor_poll =
        sensorManager.poll(currentData, storage, pressureHistory, co2_asc_enabled);
    uiController.onSensorPoll(sensor_poll);
    chartsHistory.update(currentData, storage);
    chartsRuntimeState.update(chartsHistory);
    webRuntimeState.update(currentData, sensorManager.isWarmupActive(), fanControl);
```

**逐段解析**：每次循环先轮询传感器，`PollResult` 携带 `data_changed`/`warmup_changed` 标志位，避免下游消费者在数据未变时做无谓刷新。图表历史按 5 分钟步长（`CHART_HISTORY_STEP_MS`）采样，24 小时共 288 个样本点。

```cpp
    if (BootPolicy::markStable(now, boot_start_ms, Config::SAFE_BOOT_STABLE_MS,
                               boot_stable, boot_count, safe_boot_stage)) {
        OtaRollback::markValidIfPending("stable_boot");
    }
```

**逐段解析**：稳定启动判定——若距 `boot_start_ms` 已过 60 秒且未崩溃，`markStable` 清零启动计数并返回 true，随后 `OtaRollback::markValidIfPending` 将待定的 OTA 固件标记为有效，防止下次启动回滚。这是"防变砖"的关键：新固件必须运行满 60 秒才被确认，否则崩溃重启会自动回滚到上一版本。

### 4.2 Safe Boot 防崩溃回滚策略（BootPolicy.cpp）

这是 Project Aura 可靠性设计的精华。`BootPolicy::apply` 根据崩溃重启次数决定启动动作：

```cpp
StorageManager::BootAction BootPolicy::apply(bool crash_reset,
                                             uint32_t &boot_count,
                                             uint32_t &safe_boot_stage,
                                             uint8_t max_reboots) {
    if (!crash_reset) {
        boot_count = 0;
        safe_boot_stage = 0;
        return StorageManager::BootAction::Normal;
    }

    if (boot_count < UINT32_MAX) {
        boot_count++;
    }

    if (boot_count >= max_reboots) {
        if (safe_boot_stage == 0) {
            safe_boot_stage = 1;
            return StorageManager::BootAction::SafeRollback;
        }
        return StorageManager::BootAction::SafeFactoryReset;
    }

    return StorageManager::BootAction::Normal;
}
```

**逐行解析**：
- 第 1 个 `if`：非崩溃复位（如正常上电或深度睡眠唤醒），重置计数，正常启动。
- `boot_count++`：每次崩溃重启递增计数。
- `boot_count >= max_reboots`（默认 5 次）：连续崩溃 5 次触发保护。`safe_boot_stage == 0` 时进入第一阶段 `SafeRollback`——回滚到上一已知良好固件版本。若回滚后仍崩溃（`safe_boot_stage == 1`），进入第二阶段 `SafeFactoryReset`——恢复出厂配置。
- 这种两级保护确保即使新固件有严重 bug，设备也不会彻底变砖。

```cpp
bool BootPolicy::markStable(uint32_t now_ms, uint32_t boot_start_ms,
                            uint32_t stable_ms, bool &boot_stable,
                            uint32_t &boot_count, uint32_t &safe_boot_stage) {
    if (boot_stable) { return false; }
    if ((now_ms - boot_start_ms) < stable_ms) { return false; }
    boot_count = 0;
    safe_boot_stage = 0;
    boot_stable = true;
    return true;
}
```

**逐行解析**：`markStable` 仅在启动后达到稳定阈值（60 秒）时执行一次。一旦标记稳定，清零所有崩溃计数和阶段，后续崩溃重新从零计数。`boot_stable` 引用参数确保只触发一次。

### 4.3 传感器管理器——多传感器协调与数据校验（SensorManager.cpp）

`SensorManager::begin()` 展示了 Project Aura 的硬件自动探测策略，气压传感器按优先级依次尝试：

```cpp
void SensorManager::begin(StorageManager &storage, float temp_offset, float hum_offset) {
    sen66_.begin();
    sen66_.setOffsets(temp_offset, hum_offset);
    sen66_.loadVocState(storage);
    sen66_start_attempts_ = 0;
    sen66_retry_exhausted_logged_ = false;

    bmp580_.begin();
    if (bmp580_.start()) {
        pressure_sensor_ = PRESSURE_BMP58X;
        Logger::log(Logger::Info, "Sensors", "%s OK", bmp580_.variantLabel());
    } else {
        bmp3xx_.begin();
        if (bmp3xx_.start()) {
            pressure_sensor_ = PRESSURE_BMP3XX;
            // ...
        } else {
            dps310_.begin();
            if (dps310_.start()) {
                pressure_sensor_ = PRESSURE_DPS310;
            } else {
                pressure_sensor_ = PRESSURE_NONE;
                LOGW("Sensors", "Pressure sensor not found");
            }
        }
    }
```

**逐段解析**：SEN66 作为主传感器优先初始化，并从存储加载上次的 VOC 算法状态（VOC index 需要历史基线，断电后需恢复）。气压传感器采用三级降级探测：先 BMP580（最新最准）→ 失败则 BMP3xx（BMP388/390）→ 再失败则 DPS310 → 全无则标记 `PRESSURE_NONE`。这种设计让同一份固件兼容多种硬件配置，用户无需重新编译。

`begin()` 中 HCHO 传感器的探测逻辑更复杂，因为它需要区分 SFA30 和 SFA40 两种型号，并处理热重启场景：

```cpp
    hcho_sensor_type_ = HCHO_SENSOR_NONE;
    const bool hcho_warm_restart = (boot_reset_reason != ESP_RST_POWERON);
    bool sfa30_identified = false;
    sfa30_.begin();
    sfa40_.begin();

    if (hcho_warm_restart) {
        sfa30_identified = sfa30_.probe();
        if (sfa30_identified) {
            sfa30_.start();
            if (sfa30_.status() == Sfa30::Status::Ok) {
                hcho_sensor_type_ = HCHO_SENSOR_SFA30;
            }
        }
    }
```

**逐段解析**：`boot_reset_reason != ESP_RST_POWERON` 判断是否为热重启（如 OTA 后软重启）。热重启时先探测 SFA30——因为 SFA40 在某些情况下会响应 SFA30 的地址造成误识别，热重启时 SFA30 可能已在运行。仅在冷启动或 SFA30 未识别时才尝试 SFA40，最后再回退到 SFA30。这体现了对 I2C 设备识别边界情况的深入理解。

`apply_sanity_filters()` 是数据质量的关键防线，对每个传感器读数进行范围校验：

```cpp
bool apply_sanity_filters(SensorData &data, float hcho_min_ppb, float hcho_max_ppb) {
    bool changed = false;

    if (data.temp_valid &&
        (!isfinite(data.temperature) ||
         data.temperature < Config::SEN66_TEMP_MIN_C ||
         data.temperature > Config::SEN66_TEMP_MAX_C)) {
        data.temp_valid = false;
        data.temperature = 0.0f;
        changed = true;
    }
    // ... 湿度、CO2、VOC、NOx、PM 各项同理校验

    if (data.pm25_valid) {
        if (!isfinite(data.pm25)) {
            data.pm25_valid = false;
            data.pm25 = 0.0f;
            changed = true;
        } else {
            float clamped = clampf(data.pm25, Config::SEN66_PM_MIN_UGM3, Config::SEN66_PM_MAX_UGM3);
            if (clamped != data.pm25) {
                data.pm25 = clamped;
                changed = true;
            }
        }
    }
```

**逐段解析**：每个传感器读数先检查 `isfinite()`（排除 NaN/Inf），再检查是否在数据手册规定的硬限值内（如温度 -10~60°C，PM 0~999 μg/m³）。超限则标记 `valid = false` 并清零，而非传递错误值给 UI 和 MQTT。`clampf` 对可接受但越界的值做钳位。这种防御性编程确保即使传感器瞬时故障也不会污染历史数据。

`log_air_metric_transition()` 实现了空气质量等级变化的智能日志，只在等级跨越时记录：

```cpp
void log_air_metric_transition(const char *name, const char *unit,
                               float value, bool valid,
                               float good_max, float moderate_max, float bad_max,
                               bool good_inclusive, const char *value_fmt,
                               AlertBand &previous_band) {
    if (!valid || !isfinite(value)) {
        previous_band = AlertBand::Unknown;
        return;
    }

    const AlertBand current_band =
        classify_upper_band(value, good_max, moderate_max, bad_max, good_inclusive);
    if (current_band == previous_band) {
        return;
    }
    // ... 仅在等级变化时记录 LOGW/LOGI
```

**逐段解析**：`AlertBand` 分为 Good/Moderate/Bad/Critical 四级。`classify_upper_band` 根据阈值（如 CO2：< 800 绿 / < 1000 黄 / < 1500 橙 / ≥ 1500 红）归类。`previous_band` 静态变量记忆上次等级，仅当跨级时输出日志——避免每秒刷屏，同时让用户能在串口日志中看到"CO2 升高到橙区"这类关键事件。

### 4.4 SEN66 I2C 驱动——协议级实现（Sen66.cpp）

SEN66 采用 Sensirion 的 I2C 命令协议：每个命令字为 16 位，数据字以"高字节+低字节+CRC8"三元组传输。`readWords()` 是底层读取原语：

```cpp
bool Sen66::readWords(uint16_t cmd, uint16_t *out, size_t words, uint32_t delay_ms) {
    if (I2C::write_cmd(Config::SEN66_ADDR, cmd, nullptr, 0) != ESP_OK) {
        return false;
    }
    delay(delay_ms);
    const size_t bytes = words * 3;
    uint8_t buf[27];
    if (bytes > sizeof(buf)) {
        return false;
    }
    if (I2C::read_bytes(Config::SEN66_ADDR, buf, bytes) != ESP_OK) {
        return false;
    }
    for (size_t i = 0; i < words; ++i) {
        const uint8_t *p = &buf[i * 3];
        if (I2C::crc8(p, 2) != p[2]) {
            return false;
        }
        out[i] = (static_cast<uint16_t>(p[0]) << 8) | p[1];
    }
    return true;
}
```

**逐行解析**：
- 先发送 16 位命令字（`write_cmd`），不带参数。
- `delay(delay_ms)`：SEN66 命令需要等待测量完成，不同命令延时不同（读取数值 20ms，FRC 校准 500ms）。
- 每个数据字占 3 字节：2 字节数据 + 1 字节 CRC8。`words * 3` 计算总字节数。
- 循环校验每个字的 CRC8——Sensirion 的 CRC8 多项式为 0x31，初值 0xFF。任一字校验失败立即返回 false，保证数据完整性。
- 大端序组装：高字节在前 `(p[0] << 8) | p[1]`。

`readValues()` 解析 SEN66 的 9 字测量数据块，展示原始值到工程量的转换：

```cpp
bool Sen66::readValues(SensorData &out) {
    uint16_t words[9];
    if (!readWords(Config::SEN66_CMD_READ_VALUES, words, 9, Config::SEN66_CMD_DELAY_MS)) {
        return false;
    }

    const uint16_t pm1_raw = words[0];
    const uint16_t pm25_raw = words[1];
    // ... words[2-3] = pm4, pm10
    const int16_t rh_raw = static_cast<int16_t>(words[4]);
    const int16_t t_raw = static_cast<int16_t>(words[5]);
    const int16_t voc_raw = static_cast<int16_t>(words[6]);
    const int16_t nox_raw = static_cast<int16_t>(words[7]);
    const uint16_t co2_raw = words[8];

    out.pm25_valid = (pm25_raw != 0xFFFF);
    if (out.pm25_valid) {
        out.pm25 = pm25_raw / 10.0f;
    }
    // ...
    out.hum_valid = (rh_raw != 0x7FFF);
    if (out.hum_valid) {
        out.humidity = (rh_raw / 100.0f) + hum_offset_;
    }

    out.temp_valid = (t_raw != 0x7FFF);
    if (out.temp_valid) {
        float temp_correction = desiredTempCorrectionC();
        if (temp_offset_hw_active_) {
            temp_correction -= temp_offset_hw_value_;
        }
        out.temperature = (t_raw / 200.0f) + temp_correction;
    }
```

**逐行解析**：
- 9 个字依次为：PM1、PM2.5、PM4、PM10、湿度、温度、VOC、NOx、CO2。
- `0xFFFF`（无符号）和 `0x7FFF`（有符号）是 SEN66 的"无效值"标记，检测到则标记 `valid = false`。
- **PM 缩放**：`/ 10.0f`，原始值单位为 0.1 μg/m³。
- **湿度缩放**：`/ 100.0f`，原始值单位为 0.01 %RH，再加上用户偏移 `hum_offset_`。
- **温度缩放**：`/ 200.0f`，原始值单位为 0.005 °C。温度补偿较复杂：`desiredTempCorrectionC()` 返回用户设定偏移减去基础偏移（`BASE_TEMP_OFFSET = 2.4°C`，因为 SEN66 自身有 2.4°C 的板载温升）。`temp_offset_hw_active_` 表示偏移已写入传感器硬件寄存器，此时软件层需减去已应用的硬件值避免双重补偿。

CO2 平滑算法 `smoothCo2()` 是一个带突变检测的滑动平均滤波器：

```cpp
int Sen66::smoothCo2(int new_val) {
    if (co2_first_) {
        for (int i = 0; i < 5; i++) {
            co2_readings_[i] = new_val;
        }
        co2_first_ = false;
    }

    long sum = 0;
    for (int i = 0; i < 5; i++) {
        sum += co2_readings_[i];
    }
    int avg = static_cast<int>(sum / 5);

    if (abs(new_val - avg) > 150) {
        for (int i = 0; i < 5; i++) {
            co2_readings_[i] = new_val;
        }
        return new_val;
    }

    co2_readings_[co2_idx_] = new_val;
    co2_idx_ = (co2_idx_ + 1) % 5;

    sum = 0;
    for (int i = 0; i < 5; i++) {
        sum += co2_readings_[i];
    }
    return static_cast<int>(sum / 5);
}
```

**逐行解析**：
- 首次采样用新值填满整个 5 元素缓冲区，避免前几次平均失真。
- 计算当前 5 次平均值。
- **突变检测**：若新值与均值偏差超过 150ppm（如开门通风导致 CO2 骤变），立即用新值覆盖整个缓冲区并返回——这避免了滑动平均对快速变化的滞后。
- 正常情况下环形写入新值（`co2_idx_ = (co2_idx_ + 1) % 5`），返回新的 5 点均值。这种设计在平滑噪声的同时保留了响应速度。

SEN66 的启动序列 `start()` 处理温度补偿、VOC 状态恢复、ASC（自动基线校准）和测量启动：

```cpp
bool Sen66::start(bool asc_enabled) {
    busy_ = true;
    if (!forceIdle()) {
        ok_ = false;
        measuring_ = false;
        busy_ = false;
        return false;
    }
    if (!applyTempOffsetParams()) {
        LOGW("SEN66", "temp offset set failed");
    }
    if (voc_state_valid_) {
        if (!setVocState(voc_state_, sizeof(voc_state_))) {
            LOGW("SEN66", "VOC state restore failed");
        }
    }
    if (asc_enabled && asc_default_known_) {
        Logger::log(Logger::Info, "SEN66", "ASC enabled (default after reset)");
    } else if (!setAscRaw(asc_enabled)) {
        Logger::log(Logger::Warn, "SEN66", "ASC set failed (%s)",
                    asc_enabled ? "enable" : "disable");
    }
    if (!startMeasurement()) {
        ok_ = false;
        busy_ = false;
        return false;
    }
    ok_ = true;
    busy_ = false;
    return true;
}
```

**逐行解析**：
- `forceIdle()`：热重启后传感器可能仍在测量状态，先强制停止（发送 STOP 命令，最多重试 3 次，失败则硬件复位）。
- `applyTempOffsetParams()`：将温度补偿写入 SEN66 的硬件寄存器（命令 `0x60B2`），让传感器自身输出已补偿的温度。
- `voc_state_valid_`：若存储中有上次保存的 VOC 状态（8 字节），恢复到传感器——VOC index 算法依赖历史基线，断电恢复可避免重新预热学习。
- ASC（Automatic Self-Calibration）：CO2 的自动基线校准。`asc_default_known_` 仅在硬复位后为 true（ASC 默认启用），此时无需重复写入；热重启后状态未知，需显式设置。`setAscRaw()` 内部带写入+读回验证+重试机制（2 次写入尝试 × 6 次验证）。

### 4.5 风扇控制与自动通风需求算法（FanControl.cpp）

`FanControl` 管理 GP8403 DAC 输出 0-10V 控制外部风机，支持手动和自动两种模式。自动模式的核心是 `evaluateAutoDemandPercent()`——根据多传感器读数计算通风需求百分比：

```cpp
uint8_t FanControl::evaluateAutoDemandPercent(const SensorData &data, bool gas_warmup) const {
    uint8_t demand = 0;

    const auto pick_percent = [&](const DacAutoSensorConfig &sensor,
                                  bool valid, float value,
                                  float green_limit, float yellow_limit,
                                  float orange_limit) -> uint8_t {
        if (!sensor.enabled || !valid) {
            return 0;
        }
        if (value < green_limit) {
            return sensor.band.green_percent;
        }
        if (value < yellow_limit) {
            return sensor.band.yellow_percent;
        }
        if (value < orange_limit) {
            return sensor.band.orange_percent;
        }
        return sensor.band.red_percent;
    };

    const bool co2_valid = data.co2_valid && data.co2 > 0;
    demand = maxPercent(demand, pick_percent(auto_config_.co2,
                                             co2_valid,
                                             static_cast<float>(data.co2),
                                             Config::AQ_CO2_GREEN_MAX_PPM,
                                             Config::AQ_CO2_YELLOW_MAX_PPM,
                                             Config::AQ_CO2_ORANGE_MAX_PPM));
```

**逐段解析**：
- `pick_percent` 是一个 lambda，按四色阈值（绿/黄/橙/红）将传感器读数映射为预设的通风百分比。例如 CO2：< 800ppm（绿）返回用户设定的绿色档百分比；800-1000（黄）；1000-1500（橙）；≥ 1500（红）。
- `maxPercent(demand, ...)`：取所有已启用传感器的最大需求——"木桶效应"，任一指标超标即触发对应级别的通风。
- `gas_warmup` 参数：VOC/NOx 在预热期（5 分钟）读数不可靠，调用方会跳过这两项（见后文 `voc_valid = !gas_warmup && ...`）。

自动模式的执行逻辑在 `poll()` 中：

```cpp
    if (mode_ == Mode::Auto && available_ && !manual_override_active_ && !auto_resume_blocked_) {
        uint8_t demand_percent = 0;
        if (auto_config_.enabled && sensor_data != nullptr) {
            demand_percent = evaluateAutoDemandPercent(*sensor_data, gas_warmup);
        }
        const uint16_t target_mv = percentToMillivolts(demand_percent);

        if (target_mv == 0) {
            if (running_ || !output_known_ || output_mv_ != Config::DAC_SAFE_ERROR_MV) {
                if (!applyOutputMillivolts(Config::DAC_SAFE_ERROR_MV)) {
                    handleDacFault("auto stop write failed");
                    publishSnapshot();
                    return;
                }
                applyStopState(true);
            }
        } else {
            if (!running_ || output_mv_ != target_mv) {
                if (!applyOutputMillivolts(target_mv)) {
                    handleDacFault("auto level write failed");
                    publishSnapshot();
                    return;
                }
            }
            running_ = true;
            output_known_ = true;
            output_mv_ = target_mv;
            stop_at_ms_ = 0;
        }
    }
```

**逐段解析**：
- 仅在自动模式、DAC 可用、无手动覆盖、自动未被阻断时执行。
- 需求为 0% 时写入安全电压（0mV）并停止；非 0% 时按需写入对应电压。`applyOutputMillivolts` 内部带一次重试。
- `manual_override_active_`：用户手动操作后置 true，阻止自动模式立即抢回控制权，需用户显式"重新启用自动"才恢复。
- `auto_resume_blocked_`：自动模式启动后有 15 秒延迟（`DAC_AUTO_BOOT_RESUME_DELAY_MS`）才接管，避免开机瞬间传感器未稳定就满负荷通风。

`FanControl` 的线程安全设计值得注意——它使用 FreeRTOS 互斥锁保护共享状态：

```cpp
void FanControl::ensureSyncPrimitives() {
    if (sync_mutex_ == nullptr) {
        sync_mutex_ = xSemaphoreCreateMutex();
    }
}

bool FanControl::lockSync() const {
    if (sync_mutex_ == nullptr) {
        return true;
    }
    return xSemaphoreTake(sync_mutex_, portMAX_DELAY) == pdTRUE;
}
```

**逐行解析**：`sync_mutex_` 懒创建（首次使用时创建）。UI 线程（Core 1）通过 `setMode()`/`requestStart()` 等接口写入 `pending_commands_`，`poll()` 在主循环（Core 0）中 `drainPendingCommands()` 读取并清空——命令队列模式避免了对 DAC 状态的直接跨核竞争。`portMAX_DELAY` 表示无限等待获取锁。

### 4.6 GP8403 DAC 驱动——电压到原始值的转换（Gp8403.cpp）

GP8403 是一个 12 位双通道 I2C DAC，输出范围可配置 5V 或 10V。`writeChannelMillivolts()` 实现毫伏到 12 位原始值的转换：

```cpp
bool Gp8403::writeChannelMillivolts(uint8_t channel, uint16_t millivolts) {
    if (millivolts < Config::DAC_VOUT_MIN_MV) {
        millivolts = Config::DAC_VOUT_MIN_MV;
    } else if (millivolts > Config::DAC_VOUT_FULL_SCALE_MV) {
        millivolts = Config::DAC_VOUT_FULL_SCALE_MV;
    }

    if (Config::DAC_VOUT_FULL_SCALE_MV == 0) {
        return false;
    }

    const uint32_t numerator = static_cast<uint32_t>(millivolts) * 4095u +
                               (Config::DAC_VOUT_FULL_SCALE_MV / 2u);
    const uint16_t raw12 = static_cast<uint16_t>(numerator / Config::DAC_VOUT_FULL_SCALE_MV);
    return writeChannelRaw12(channel, raw12);
}

bool Gp8403::writeChannelRaw12(uint8_t channel, uint16_t raw12) {
    const uint8_t reg = channelRegister(channel);
    if (reg == 0) {
        return false;
    }
    if (raw12 > 0x0FFF) {
        raw12 = 0x0FFF;
    }

    const uint16_t packed = static_cast<uint16_t>(raw12 << 4);
    const uint8_t payload[2] = {
        static_cast<uint8_t>(packed & 0xFF),
        static_cast<uint8_t>((packed >> 8) & 0xFF),
    };
    return writeRegister(reg, payload, sizeof(payload));
}
```

**逐行解析**：
- 先钳位到 0-10000mV 范围。
- **12 位 DAC 转换公式**：`raw12 = (millivolts * 4095 + 5000) / 10000`。加 `DAC_VOUT_FULL_SCALE_MV / 2`（5000）实现四舍五入，提高精度。用 `uint32_t` 中间变量避免 16 位溢出。
- `raw12 << 4`：GP8403 的寄存器是 16 位对齐，12 位数据需左移 4 位到高位。低字节在前（小端序）写入。
- `channelRegister()` 把通道号映射到寄存器地址（通道 0 → 0x02，通道 1 → 0x04）。

### 4.7 MQTT 集成与 Home Assistant 自动发现（MqttManager.cpp）

`publishDiscovery()` 是 Home Assistant 集成的核心，它向 `homeassistant/*/config` 主题发布所有实体的发现配置：

```cpp
void MqttManager::publishDiscovery(const MqttRuntimeSnapshot &runtime) {
    if (!mqtt_discovery_ || mqtt_discovery_sent_ || !mqtt_connected_) {
        return;
    }
    const auto clear_discovery = [&](const char *component, const char *object_id) {
        char topic[kTopicBufferSize];
        build_discovery_topic(topic, sizeof(topic), component, mqtt_device_id_, object_id);
        publishMessage(topic, "", true);
    };
    // 清理旧版本遗留的 retained 实体
    char legacy_topic[kTopicBufferSize];
    build_discovery_topic(legacy_topic, sizeof(legacy_topic), "sensor", mqtt_device_id_, "pm4_0");
    publishMessage(legacy_topic, "", true);
```

**逐段解析**：
- `mqtt_discovery_sent_` 标志确保发现消息每次连接只发一次。
- `clear_discovery` lambda：向发现主题发布空字符串（retain=true），清除旧版固件遗留的实体配置——这是固件升级时的实体清理策略，避免 HA 中残留废弃实体。

随后逐个注册所有传感器实体：

```cpp
    publishDiscoverySensor("temperature", "Temperature", "\\u00b0C",
                           "temperature", "measurement", "{{ value_json.temp }}", "");
    publishDiscoverySensor("co2", "CO2", "ppm",
                           "carbon_dioxide", "measurement", "{{ value_json.co2 }}", "");
    publishDiscoverySensor("pm25", "PM2.5", "\\u00b5g/m\\u00b3",
                           "pm25", "measurement", "{{ value_json.pm25 }}", "");
    publishDiscoverySensor("voc_index", "VOC Index", "index",
                           "", "measurement", "{{ value_json.voc_index }}", "mdi:blur");
    publishDiscoverySensor("hcho", "HCHO", "ppb",
                           "volatile_organic_compounds_parts", "measurement",
                           "{{ value_json.hcho }}", "mdi:flask-outline");
```

**逐行解析**：每个 `publishDiscoverySensor` 调用注册一个 HA 传感器实体，参数依次为：object_id、显示名、单位、device_class（HA 内置图标和单位推导）、state_class（measurement=瞬时测量值）、value_template（从 state JSON 提取字段）、icon。例如 CO2 用 `carbon_dioxide` device_class，PM2.5 用 `pm25` device_class，HA 会自动渲染对应的图标和趋势图。

可选的通风控制实体仅在 DAC 存在时注册：

```cpp
    if (runtime.fan.present) {
        publishDiscoverySwitch("fan_auto", "Ventilation Auto", "{{ value_json.fan_auto }}",
                               "mdi:fan-auto");
        publishDiscoverySelect("fan_mode", "Ventilation Mode",
                               "{{ value_json.fan_control_mode }}",
                               kFanModeOptions, ..., "mdi:fan-cog");
        publishDiscoveryNumber("fan_manual_percent", "Ventilation Speed",
                               "{{ value_json.fan_manual_percent }}",
                               10, 100, 10, "slider", "mdi:fan");
        publishDiscoveryBinarySensor("fan_fault", "Ventilation Fault",
                                     "{{ value_json.fan_fault }}", "problem", "mdi:fan-alert");
    } else {
        clear_discovery("fan", "fan");
        clear_discovery("switch", "fan_auto");
        // ... 清除所有通风实体
    }
```

**逐段解析**：通风控制注册了开关（Auto/Manual/Stop）、下拉选择（模式）、数字滑块（速度 10-100%，步进 10）、故障二值传感器。无 DAC 时清除所有相关实体，保持 HA 界面整洁。

命令处理 `handleIncomingMessage()` 解析 HA 下发的控制指令：

```cpp
void MqttManager::handleIncomingMessage(const char *topic, const uint8_t *payload, size_t length) {
    // ... 提取 base_topic 和命令后缀
    String cmd = suffix.substring(strlen("/command/"));
    bool is_on = payloadIsOn(msg);
    bool is_off = payloadIsOff(msg);
    MqttPendingCommands pending_update;
    bool has_pending_update = false;

    if (cmd == "night_mode") {
        if (auto_night_enabled) {
            LOGI("MQTT", "night mode ignored (auto night enabled)");
            return;
        }
        if (is_on || is_off) {
            pending_update.night_mode_value = is_on;
            pending_update.night_mode = true;
            has_pending_update = true;
        }
    } else if (cmd == "alert_blink") {
        // ...
    } else if (cmd == "fan_manual_percent") {
        uint8_t percent = 0;
        if (parse_uint8_in_range(msg, 10, 100, percent) && (percent % 10u) == 0u) {
            pending_update.fan_manual_speed_value = static_cast<uint8_t>(percent / 10u);
            pending_update.fan_manual_speed = true;
            has_pending_update = true;
        }
    }
```

**逐段解析**：命令主题格式为 `<base>/command/<cmd>`。`night_mode` 命令检查 `auto_night_enabled`——若自动夜间模式已启用，手动命令被忽略，避免冲突。`fan_manual_percent` 严格校验：值必须在 10-100 范围内且为 10 的倍数（`percent % 10u == 0u`），转换为 1-10 档速度。所有命令先写入 `pending_update` 队列，由主循环统一应用，避免在 MQTT 回调线程中直接操作硬件。

---

## 五、API 使用指南

Project Aura 在设备本地暴露一组 REST API，供仪表盘和外部集成使用。

### 获取实时状态

```bash
# 获取所有传感器实时数据、网络状态、通风状态
curl http://aura-1a2b3c.local/api/state
```

返回 JSON 示例（节选）：

```json
{
  "network": { "mode": "STA", "ip": "192.168.1.42", "hostname": "aura-1a2b3c" },
  "temp": 23.4, "temp_valid": true,
  "humidity": 45.2, "humidity_valid": true,
  "co2": 612, "co2_valid": true,
  "pm25": 8.3, "pm25_valid": true,
  "voc_index": 102, "nox_index": 15,
  "pressure": 1013.2, "pressure_valid": true,
  "fan": { "present": true, "running": false, "mode": "auto" }
}
```

### 获取历史图表数据

```bash
# 核心指标（温湿度/CO2），24小时窗口
curl "http://aura-1a2b3c.local/api/charts?group=core&window=24h"

# 颗粒物，3小时窗口
curl "http://aura-1a2b3c.local/api/charts?group=pm&window=3h"

# 气体，1小时窗口
curl "http://aura-1a2b3c.local/api/charts?group=gases&window=1h"
```

### Python 集成示例

```python
import requests
import time

AURA_HOST = "http://aura-1a2b3c.local"

def get_air_quality():
    """获取 Project Aura 实时空气质量数据"""
    resp = requests.get(f"{AURA_HOST}/api/state", timeout=10)
    resp.raise_for_status()
    data = resp.json()

    return {
        "timestamp": time.time(),
        "temperature": data.get("temp"),
        "humidity": data.get("humidity"),
        "co2_ppm": data.get("co2"),
        "pm25_ugm3": data.get("pm25"),
        "voc_index": data.get("voc_index"),
        "nox_index": data.get("nox_index"),
        "pressure_hpa": data.get("pressure"),
    }

def set_night_mode(enabled: bool):
    """通过 MQTT 命令主题切换夜间模式（需 MQTT 已配置）"""
    import paho.mqtt.client as mqtt
    client = mqtt.Client()
    client.connect("your-mqtt-broker.local", 1883)
    client.publish("project_aura/command/night_mode", "ON" if enabled else "OFF")
    client.disconnect()

if __name__ == "__main__":
    aq = get_air_quality()
    print(f"CO2: {aq['co2_ppm']} ppm | PM2.5: {aq['pm25_ugm3']} μg/m³ | "
          f"VOC: {aq['voc_index']} | Temp: {aq['temperature']}°C")
```

### Home Assistant MQTT 控制

```bash
# 切换夜间模式
mosquitto_pub -h broker.local -t "project_aura/command/night_mode" -m "ON"

# 设置通风手动速度（50%）
mosquitto_pub -h broker.local -t "project_aura/command/fan_manual_percent" -m "50"

# 重启设备
mosquitto_pub -h broker.local -t "project_aura/command/restart" -m "PRESS"
```

---

## 六、编译与部署

### 环境搭建

| 项 | 要求 |
|----|------|
| 开发框架 | PlatformIO Core 或 VSCode + PlatformIO 扩展 |
| Arduino ESP32 Core | 3.1.1（ESP-IDF 5.3.x） |
| LVGL | 8.4.0 |
| 目标板 | esp32-s3-devkitc-1（16MB Flash） |
| Flash 类型 | QIO + OPI PSRAM（`qio_opi`） |
| 分区表 | `partitions_16MB_littlefs.csv`（LittleFS 文件系统） |

### 编译配置（platformio.ini 关键项）

```ini
[env:project_aura]
platform = .../platform-espressif32/releases/download/53.03.11/...
board = esp32-s3-devkitc-1
framework = arduino
board_upload.flash_size = 16MB
board_build.partitions = partitions_16MB_littlefs.csv
board_build.filesystem = littlefs
board_build.arduino.memory_type = qio_opi

build_flags =
    -DCORE_DEBUG_LEVEL=1
    -DAPP_VERSION=\"1.1.5\"
    -DLV_CONF_INCLUDE_SIMPLE
    -DLV_COLOR_16_SWAP=0

lib_deps =
    https://github.com/esp-arduino-libs/ESP32_Display_Panel.git
    https://github.com/lvgl/lvgl.git#v8.4.0
    bblanchon/ArduinoJson@^7.0.0
```

> 构建前会执行 4 个预处理脚本：`set_build_id.py`（生成唯一构建 ID）、`generate_dashboard_gzip.py`/`generate_dac_gzip.py`/`generate_theme_gzip.py`（将 Web 资产预压缩为 gzip，减少运行时内存占用）。

### 编译与烧录步骤

```bash
# 1. 克隆仓库
git clone https://github.com/21cncstudio/project_aura.git
cd project_aura

# 2.（可选）配置编译时默认 Wi-Fi/MQTT
cp include/secrets.h.example include/secrets.h
# 编辑 include/secrets.h 填入 SSID/密码/MQTT 信息

# 3. 编译固件
pio run -e project_aura

# 4. 烧录固件（通过 USB-C 数据口）
pio run -e project_aura -t upload

# 5. 烧录 LittleFS 文件系统（Web 仪表盘和翻译资源，首次必须执行）
pio run -e project_aura -t uploadfs

# 6. 串口监视器
pio device monitor -b 115200
```

> **首次烧录也可用浏览器 Web 安装器**：访问 https://aura.21cncstudio.com/ ，用兼容浏览器连接 Waveshare 板的数据 USB-C 口即可一键烧录，无需安装任何工具链。后续更新可通过设备本地仪表盘的 OTA 页面上传 `.bin` 文件。

### 首次启动配置

1. 设备创建热点 `Aura-XXXXXX-AP`（回退名 `ProjectAura-Setup`）。
2. 连接热点，浏览器打开 `http://192.168.4.1` 配置 Wi-Fi。
3. 保存后约 15 秒完成 AP→STA 切换。
4. 通过 `http://aura-XXXXXX.local/` 或设备屏幕显示的 IP 访问仪表盘。

---

## 七、项目亮点与适用场景

### 技术亮点

1. **硬件全自动探测**：同一份固件兼容 BMP580/388/390/DPS310 气压、PCF8523/DS3231 RTC、SFA30/SFA40 甲醛、5 种 DFR 电化学气体，无需重新编译或改配置。
2. **两级 Safe Boot 防变砖**：崩溃 5 次先回滚固件，再崩溃则恢复出厂，配合 60 秒稳定启动判定和 OTA 回滚保护，OTA 升级几乎不可能变砖。
3. **协议级传感器驱动**：SEN66 驱动完整实现了 Sensirion I2C 命令协议（CRC8 校验、数据就绪轮询、VOC 状态持久化、ASC 自动校准、FRC 强制校准、温度硬件补偿），非简单调库。
4. **智能通风需求算法**：多传感器四色阈值取最大值的自动通风控制，带手动覆盖阻断、开机延迟接管、突变响应，把空气质量监测从"只看"变成"可控制"。
5. **线程安全设计**：FreeRTOS 互斥锁 + 命令队列模式，UI（Core 1）与传感器/DAC（Core 0）安全分离；专用 Core 0 重启任务避免 IPC 栈 panic。
6. **完整 Home Assistant 集成**：MQTT 自动发现注册 30+ 实体（传感器/开关/下拉/滑块/事件），双向控制，开箱即用。
7. **离线优先**：AP 配网和本地仪表盘完全离线可用，无云端依赖，mDNS 本地访问。

### 适用场景

| 场景 | 适用传感器组合 |
|------|---------------|
| 家庭/办公室空气监测 | SEN66 + 气压 + RTC |
| 装修污染追踪 | SEN66 + SFA30/40（HCHO 甲醛） |
| 车库/工坊燃烧安全 | SEN66 + SEN0466（CO） |
| 农场/动物房 | SEN66 + SEN0469（NH3） |
| 实验室/打印室 | SEN66 + 可选气体 + DAC 通风控制 |
| 智能家居联动 | SEN66 + MQTT + Home Assistant + DAC |

### 二次开发方向

- 替换/新增其他 I2C 传感器：参照 `src/drivers/` 中的驱动模式（begin/probe/poll/takeNewData）实现新驱动，在 `SensorManager::begin()` 中加入探测链。
- 自定义通风策略：修改 `evaluateAutoDemandPercent()` 的阈值逻辑或新增传感器参与计算。
- 扩展 Web API：在 `src/web/` 中参照 `WebChartsApiHandlers` 模式新增路由。
- 增加语言：在 `src/ui/strings/` 添加 `UiStrings.xx.inc` 并在 `Language` 枚举注册。

---

## 八、总结

Project Aura AQ 是一个罕见的"成品级"开源嵌入式项目。它没有停留在"把传感器读数打印到串口"的原型阶段，而是构建了从硬件自动探测、协议级驱动、数据校验、Safe Boot 防变砖、线程安全通风控制到 MQTT/Home Assistant 双向集成的完整工程链路。

从源码层面看，三个设计尤其值得学习：一是 SEN66 驱动对 Sensirion I2C 协议的完整实现（CRC8、VOC 状态持久化、ASC 带验证的写入重试），二是 `FanControl` 的命令队列 + 互斥锁线程安全模型，三是 `BootPolicy` 的两级崩溃回滚策略。这些都不是"能跑就行"的代码，而是经过实际产品打磨的可靠性设计。

对于想深入学习 ESP32-S3 工程级固件开发的开发者，Project Aura 的代码库是一座金矿——它的模块划分、错误处理、硬件兼容策略和测试覆盖（`test/` 目录有 69 个测试子目录）都值得仔细研读。配合 16MB Flash、LVGL 触摸 UI 和本地 Web 仪表盘，它也是少有的"固件 + 硬件 + 外壳"三位一体的开源硬件项目。

---

📝 作者：蔡浩宇（jun-chy） ｜ 📅 日期：2026-07-08 ｜ 🔗 项目地址：[https://github.com/21cncstudio/project_aura](https://github.com/21cncstudio/project_aura)
