# Project Aura：ESP32-S3 全功能空气质量监测站——深度源码解析

> **日期**：2026-06-28  
> **分类**：ESP32-S3 · 物联网 / 环境监测  
> **Star 数**：676 ⭐  
> **作者**：蔡浩宇（jun-chy）

---

## 一、项目链接

| 属性 | 信息 |
|---|---|
| **GitHub 地址** | https://github.com/21cncstudio/project_aura |
| **作者** | 21cncstudio（Volodymyr Papush） |
| **许可证** | GPL-3.0-or-later（含商业授权选项） |
| **最近更新** | 2026-06-27 |
| **创建时间** | 2026-01-22 |
| **主要语言** | C++（Arduino-ESP32 框架） |
| **Fork 数** | 59 |
| **仓库大小** | 51.6 MB |
| **标签** | esp32, esp32-s3, lvgl, mqtt, home-assistant, air-quality, sen66, iot, smart-home |

---

## 二、项目简介

Project Aura 是一个开源的 ESP32-S3 空气质量监测站，定位于"让创客拥有一台精致可靠的成品设备，而非裸传感器板"。它将触摸式 LVGL 图形界面、本地 Web 仪表盘（含浏览器 OTA 升级）、Wi-Fi 配网门户、可选 0-10V 风扇控制和 MQTT Home Assistant 自动发现整合为一体。

项目围绕 Waveshare ESP32-S3-Touch-LCD-4.3（800×480 触摸屏）和 Sensirion SEN66 多合一传感器构建，最小物料清单仅需核心板 + SEN66 + 5V 电源即可运行。固件支持颗粒物（PM0.5/1/2.5/4/10）、气体（CO/CO2/VOC/NOx/HCHO 及可选电化学气体）、气候（温湿度/绝对湿度/气压）的全面遥测，硬件自动探测多款 RTC 和气压传感器，具备崩溃后自动回滚的安全启动机制。

适用场景包括：室内空气质量监测、智能家居环境感知、新风系统联动控制、办公室/教室 CO2 监测、DIY 桌面气象站等。

---

## 三、核心特性一览表

| 特性类别 | 具体能力 |
|---|---|
| **颗粒物** | PM0.5 / PM1 / PM2.5 / PM4 / PM10 |
| **气体监测** | CO、CO2、VOC、NOx、HCHO + 可选电化学气体（NH3/SO2/NO2/H2S/O3） |
| **气候参数** | 温度、湿度、绝对湿度（AH）、气压 |
| **图形界面** | LVGL 触摸 UI，夜间模式、自定义主题、状态指示 |
| **Web 仪表盘** | `/dashboard` 实时状态、图表、事件、设置同步、DAC 控制、OTA 升级 |
| **配网** | AP 热点引导 + mDNS（`http://<hostname>.local`） |
| **智能家居** | MQTT 自动发现，Home Assistant 即插即用 |
| **风扇控制** | GP8403 DAC 0-10V 输出，手动/定时/自动阈值模式 |
| **硬件自探测** | PCF8523/DS3231 RTC、BMP58x/BMP3xx/DPS310 气压 |
| **安全启动** | 崩溃后自动回滚至上次良好配置 |
| **多语言** | 英/德/西/法/意/葡/荷/简体中文 |
| **离线运行** | AP 配网模式和本地仪表盘完全离线可用 |

---

## 四、硬件架构

### 4.1 系统组成图

```
        ┌──────────────────────────────────────────┐
        │           ESP32-S3 (Core 0)               │
        │                                          │
        │  ┌──────────┐  ┌──────────────────────┐  │
        │  │ AppInit  │  │   NetworkPlane        │ │
        │  │ BoardInit│  │ (独立网络任务)         │ │
        │  └────┬─────┘  └──────┬───────────────┘  │
        │       │               │                  │
        │  ┌────▼───────────────▼──────────────┐   │
        │  │  main loop (setup/loop)            │   │
        │  │  SensorManager → UI → MQTT → Fan   │   │
        │  └────┬───────────────────────────────┘   │
        │       │                                   │
        │  ┌────▼──────────────────────────────┐   │
        │  │  I2C Bus (GPIO8 SDA / GPIO9 SCL)  │   │
        │  └─┬─────┬──────┬──────┬──────┬──────┘   │
        └────┼─────┼──────┼──────┼──────┼──────────┘
             │     │      │      │      │
        ┌────▼──┐ ┌▼────┐┌▼────┐┌▼────┐┌▼──────┐
        │ SEN66 │ │SFA30││SEN  ││BMP  ││GP8403 │
        │多合一 │ │/SFA ││0466 ││580  ││DAC    │
        │传感器 │ │40   ││CO   ││气压 ││0-10V  │
        └───────┘ └─────┘└─────┘└─────┘└───┬──┘
                                          │
                                    ┌─────▼─────┐
                                    │ 外部风扇   │
                                    │ / 执行器   │
                                    └───────────┘
        ┌──────────────┐
        │  4.3" LCD    │← GT911 触摸 (I2C)
        │  800×480     │← 背光控制
        └──────────────┘
```

### 4.2 引脚配置

| 组件 | 引脚（ESP32-S3） | 说明 |
|---|---|---|
| 3V3 | 3V3 | 外部 I2C 传感器供电 |
| GND | GND | 公共地 |
| I2C SDA | GPIO 8 | 所有传感器和 DAC 共享总线 |
| I2C SCL | GPIO 9 | 所有传感器和 DAC 共享总线 |
| DAC 模拟输出 | GP8403 VOUT0 | 0-10V（由 DAC 模块生成，非 ESP32 直脚） |
| LCD/触摸 | 板载集成 | 无需外部接线 |

### 4.3 硬件清单

| 组件 | 型号 | 用途 | 必选 |
|---|---|---|---|
| 核心板 | Waveshare ESP32-S3-Touch-LCD-4.3（800×480） | 主控 + 显示 + 触摸 | 是 |
| 主传感器 | Sensirion SEN66（Adafruit 转接板） | 颗粒物/气体/温湿/气压 | 是 |
| CO 传感器 | DFRobot SEN0466 | 电化学 CO（0-1000 ppm） | 否 |
| 甲醛传感器 | Sensirion SFA30 或 SFA40 | HCHO（0-1000 ppb） | 否 |
| 可选气体 | DFRobot SEN0469/70/71/67/72 | NH3/SO2/NO2/H2S/O3（选一） | 否 |
| 气压传感器 | Adafruit BMP580 或 DPS310 | 高精度气压 | 否（自探测） |
| RTC | Adafruit PCF8523 | 实时时钟 | 否（自探测） |
| DAC 输出 | DFRobot GP8403（0-10V） | 风扇/执行器控制 | 否 |

---

## 五、固件架构

### 5.1 文件结构

| 目录/文件 | 职责 |
|---|---|
| `src/main.cpp` | Arduino 入口（setup/loop），全局对象实例化 |
| `src/core/AppInit.cpp` | 启动编排：启动状态处理、I2C 恢复、管理器初始化 |
| `src/core/BoardInit.cpp` | 显示板/背光/触摸硬件初始化 |
| `src/core/BootPolicy.cpp` | 安全启动策略（崩溃计数、回滚、工厂复位） |
| `src/core/Watchdog.cpp` | 任务看门狗 |
| `src/core/NetworkPlane.cpp` | 独立网络任务（Wi-Fi/MQTT/Web 并发） |
| `src/modules/SensorManager.cpp` | 传感器轮询调度与数据聚合 |
| `src/modules/FanControl.cpp` | 风扇/DAC 控制（手动/定时/自动模式） |
| `src/modules/MqttManager.cpp` | MQTT 连接、发布、Home Assistant 发现 |
| `src/modules/StorageManager.cpp` | LittleFS 配置持久化 |
| `src/modules/TimeManager.cpp` | RTC/NTP 时间管理 |
| `src/drivers/Sen66.cpp` | Sensirion SEN66 I2C 驱动 |
| `src/drivers/Gp8403.cpp` | GP8403 DAC I2C 驱动 |
| `src/drivers/Bmp580.cpp` | BMP580 气压传感器驱动 |
| `src/ui/` | LVGL 界面控制器、主题、夜间模式 |
| `src/web/` | Web 仪表盘页面与 API 路由 |
| `platformio.ini` | PlatformIO 构建配置 |
| `partitions_16MB_littlefs.csv` | 16MB Flash 分区表 |

### 5.2 模块职责说明

固件采用**分层管理器架构**。`core/` 层负责启动编排和系统策略（`AppInit` 协调启动顺序、`BootPolicy` 实现安全启动、`NetworkPlane` 在独立任务中运行网络栈）。`modules/` 层是功能管理器：`SensorManager` 调度所有传感器轮询并聚合数据，`FanControl` 根据空气质量阈值自动控制 DAC 输出，`MqttManager` 处理 MQTT 连接与 Home Assistant 发现。`drivers/` 层是各传感器的 I2C 驱动，每个驱动封装了寄存器读写和 CRC 校验。`ui/` 和 `web/` 分别负责 LVGL 触摸界面和 Web 仪表盘。

---

## 六、核心代码深度分析

### 6.1 系统启动流程（main.cpp）

Project Aura 采用 Arduino 框架的 `setup()`/`loop()` 结构，但内部组织极为精细。先看全局对象实例化：

```cpp
namespace {
using namespace Config;

SensorData currentData;
StorageManager storage;
PressureHistory pressureHistory;
ChartsHistory chartsHistory;
AuraNetworkManager networkManager;
MqttManager mqttManager;
ConnectivityRuntime connectivityRuntime;
MqttRuntimeState mqttRuntimeState;
ChartsRuntimeState chartsRuntimeState;
WebRuntimeState webRuntimeState;
NetworkCommandQueue networkCommandQueue;
WebUiBridge webUiBridge;
DisplayThresholdManager displayThresholds;
SensorManager sensorManager;
TimeManager timeManager;
ThemeManager themeManager;
BacklightManager backlightManager;
NightModeManager nightModeManager;
FanControl fanControl;
MemoryMonitor memoryMonitor;
// ...

UiContext ui_context{
    storage, networkManager, mqttManager, connectivityRuntime,
    mqttRuntimeState, webUiBridge, displayThresholds, networkCommandQueue,
    sensorManager, chartsHistory, timeManager, themeManager,
    backlightManager, nightModeManager, fanControl,
    currentData, night_mode, temp_units_c, led_indicators_enabled,
    alert_blink_enabled, co2_asc_enabled, temp_offset, hum_offset
};

UiController uiController(ui_context);
```

**逐段解析**：

- 所有管理器在匿名命名空间中作为**全局静态对象**实例化。这在 Arduino 框架中是常见模式——`setup()`/`loop()` 是自由函数，无法访问类成员，全局对象让各模块在构造时即就绪。
- `SensorData currentData` 是整个系统的**数据中枢**：所有传感器读数写入此结构，UI、MQTT、Web、FanControl 都从中读取。这种共享数据模型简化了模块间通信。
- `UiContext ui_context{...}` 使用聚合初始化将所有管理器引用打包成一个上下文结构，传给 `UiController`。这是**依赖注入**的简洁实现——UI 控制器不直接持有全局对象，而是通过 context 接收所有依赖，便于测试和重构。

`setup()` 函数编排了完整的启动序列：

```cpp
void setup()
{
    delay(3000);
    Serial.begin(115200);
    Logger::begin(Serial, static_cast<Logger::Level>(Config::LOG_LEVEL));
    Logger::setSerialOutputEnabled(Config::LOG_SERIAL_OUTPUT);
    Logger::setSensorsSerialOutputEnabled(Config::LOG_SERIAL_SENSORS_OUTPUT);
    OtaRollback::logCurrentAppState();

    memoryMonitor.begin(Config::MEM_LOG_INTERVAL_MS);
    boot_start_ms = millis();

    StorageManager::BootAction boot_action = AppInit::handleBootState();
    AppInit::recoverI2cBus(static_cast<gpio_num_t>(I2C_SDA_PIN),
                           static_cast<gpio_num_t>(I2C_SCL_PIN));

    AppInit::Context init_ctx{ /* ... 所有管理器引用 ... */ };

    AppInit::initManagersAndConfig(init_ctx, boot_action);
    auto *board = AppInit::initBoardAndPeripherals(init_ctx);
    AppInit::initLvglAndUi(init_ctx, board);
    memoryMonitor.logNow("boot");

    Watchdog::setup(TASK_WDT_TIMEOUT_MS);
    if (!safe_restart_init()) {
        LOGW("Restart", "Core0 restart task init failed; controlled restart requests will abort");
    }
    webUiBridge.setDispatchMode(WebUiBridge::DispatchMode::DeferredReply);
    network_plane_running = NetworkPlane::start(network_plane_context);
}
```

**逐段解析**：

- `delay(3000)` 开机延迟 3 秒，给串口监视器时间连接——这是嵌入式开发的实用技巧，避免错过启动日志。
- `AppInit::handleBootState()` 是**安全启动的核心**：读取复位原因（`esp_reset_reason()`），判断是否为崩溃复位。如果是，`BootPolicy` 根据 boot 计数决定回滚到上次良好配置还是工厂复位。这解决了 OTA 升级后固件崩溃导致设备"变砖"的问题——崩溃三次后自动回滚到旧固件。
- `AppInit::recoverI2cBus()` 在启动时执行 **I2C 总线恢复**：如果上一次启动异常导致某个传感器将 SDA 拉低（总线锁死），通过手动产生 9 个 SCL 时钟脉冲释放总线。这是 I2C 多设备系统的必备防御措施。
- 启动分三阶段：`initManagersAndConfig`（加载配置、初始化网络/MQTT/存储）、`initBoardAndPeripherals`（显示板、背光、RTC、传感器、风扇）、`initLvglAndUi`（LVGL 移植和 UI 创建）。每个阶段依赖前一阶段的成果，顺序严格。
- `Watchdog::setup(TASK_WDT_TIMEOUT_MS)` 设置 180 秒任务看门狗。`loop()` 中每轮 `Watchdog::kick()` 喂狗——如果主循环卡死超过 3 分钟，看门狗触发复位。
- `NetworkPlane::start()` 启动**独立网络任务**：Wi-Fi 连接、MQTT 维持、Web 服务器在专用任务中运行，不阻塞主循环。如果启动失败则回退到主循环内轮询网络（`network_plane_running = false`）。

### 6.2 主循环调度（main.cpp loop）

主循环是整个系统的节拍器，协调传感器轮询、UI 更新、网络通信和风扇控制：

```cpp
void loop()
{
    // === OTA 处理窗口 ===
    const bool ota_busy = WebHandlersIsOtaBusy();
    // ... OTA 期间暂停 LVGL 触摸读取，防止 Flash 写入与显示刷新冲突 ...

    // === 传感器轮询（主数据路径）===
    SensorManager::PollResult sensor_poll =
        sensorManager.poll(currentData, storage, pressureHistory, co2_asc_enabled);
    uiController.onSensorPoll(sensor_poll);
    chartsHistory.update(currentData, storage);
    chartsRuntimeState.update(chartsHistory);
    webRuntimeState.update(currentData, sensorManager.isWarmupActive(), fanControl);

    // === 网络处理（回退模式：网络任务未运行时在主循环内处理）===
    if (!network_plane_running) {
        networkCommandQueue.processAll(networkManager, mqttManager, connectivityRuntime);
        networkManager.poll();
        connectivityRuntime.update(networkManager, mqttManager);
    }
    AppInit::pollDeferredRuntime();

    // === 安全启动标记 ===
    const uint32_t now = millis();
    if (BootPolicy::markStable(now, boot_start_ms,
                               Config::SAFE_BOOT_STABLE_MS,
                               boot_stable, boot_count, safe_boot_stage)) {
        OtaRollback::markValidIfPending("stable_boot");
    }

    // === 时间与风扇控制 ===
    TimeManager::PollResult time_poll = timeManager.poll(now);
    mqttManager.setSystemTimeValid(timeManager.isSystemTimeValid());
    uiController.onTimePoll(time_poll);
    fanControl.poll(now, &currentData, sensorManager.isWarmupActive());

    // === 运行时状态同步 ===
    const FanControl::Snapshot fan_snapshot = fanControl.snapshot();
    webRuntimeState.update(currentData, sensorManager.isWarmupActive(), fanControl);
    mqttRuntimeState.update(currentData, fan_snapshot,
                            sensorManager.isWarmupActive(),
                            night_mode, alert_blink_enabled,
                            backlightManager.isOn(),
                            nightModeManager.isAutoEnabled());

    // === MQTT 与存储轮询 ===
    if (!network_plane_running) {
        mqttManager.poll(mqttRuntimeState);
        connectivityRuntime.update(networkManager, mqttManager);
    }
    storage.poll(now);
    memoryMonitor.poll(now);
    uiController.poll(now);
    Watchdog::kick();
    delay(10);
}
```

**逐段解析**：

- **OTA 安全窗口**：OTA 固件升级涉及 Flash 写入，而 LVGL 显示刷新也从 Flash 读取资源（字体/图片）。两者并发会导致 Cache 禁用 panic。固件通过 `ota_busy` 标志在 OTA 期间暂停 LVGL 触摸读取和显示刷新，用 `lvgl_port_request_pause()`/`lvgl_port_request_resume()` 优雅地协调。
- **传感器优先**：`sensorManager.poll()` 是每轮循环的第一项实质工作，确保数据新鲜度。返回的 `PollResult` 通知 UI 哪些数据变了，避免全量刷新。
- **数据流向清晰**：`currentData`（传感器写入）→ `chartsHistory`（历史图表）→ `webRuntimeState`（Web 状态）→ `mqttRuntimeState`（MQTT 状态）。一条数据流水线贯穿所有子系统。
- **安全启动标记** `BootPolicy::markStable()`：如果设备启动后稳定运行超过 `SAFE_BOOT_STABLE_MS`，标记当前固件为"有效"。这样下次 OTA 后如果新固件崩溃，回滚机制知道旧固件是可靠的。`OtaRollback::markValidIfPending` 实际提交这个标记到 OTA 分区。
- `fanControl.poll(now, &currentData, ...)` 根据当前空气质量数据决定 DAC 输出——如果 PM2.5 或 CO2 超过阈值，自动提高风扇电压。
- `delay(10)` 控制主循环频率约 100Hz，平衡响应速度和 CPU 占用。`Watchdog::kick()` 每轮喂狗。

### 6.3 安全启动策略（AppInit.cpp）

`handleBootState()` 是设备可靠性的第一道防线：

```cpp
StorageManager::BootAction AppInit::handleBootState() {
    esp_reset_reason_t reset_reason = esp_reset_reason();
    boot_reset_reason = reset_reason;
    boot_ui_auto_recovery_reboot = boot_consume_ui_auto_recovery_reboot();
    bool crash_reset = BootHelpers::isCrashReset(reset_reason);
    StorageManager::BootAction boot_action =
        BootPolicy::apply(crash_reset,
                          boot_count,
                          safe_boot_stage,
                          Config::SAFE_BOOT_MAX_REBOOTS);
    LOGI("Main", "Reset reason: %d (%s), boot count: %u",
         reset_reason, resetReasonName(reset_reason), boot_count);
    if (boot_action == StorageManager::BootAction::SafeRollback) {
        LOGW("Main", "SAFE BOOT: restoring last known good config");
    } else if (boot_action == StorageManager::BootAction::SafeFactoryReset) {
        LOGE("Main", "SAFE BOOT: factory reset");
    }
    return boot_action;
}
```

**逐段解析**：

- `esp_reset_reason()` 返回上次复位的精确原因：`ESP_RST_POWERON`（上电）、`ESP_RST_PANIC`（崩溃/panic）、`ESP_RST_TASK_WDT`（任务看门狗）、`ESP_RST_BROWNOUT`（掉电）等。`resetReasonName()` 将枚举转为可读字符串用于日志。
- `BootHelpers::isCrashReset()` 判断是否为异常复位（panic、看门狗、掉电等非正常上电）。这是安全启动策略的决策依据——只有异常复位才触发回滚。
- `BootPolicy::apply()` 实现分级策略：第一次崩溃尝试恢复上次良好配置（`SafeRollback`）；连续崩溃超过 `SAFE_BOOT_MAX_REBOOTS` 次则工厂复位（`SafeFactoryReset`）。`boot_count` 在 RTC 内存或 NVS 中持久化，跨复位累积。

I2C 总线恢复同样关键：

```cpp
bool AppInit::recoverI2cBus(gpio_num_t sda, gpio_num_t scl) {
    boot_i2c_recovered = BootHelpers::recoverI2CBus(sda, scl);
    if (!boot_i2c_recovered) {
        LOGW("Main", "I2C bus recovery failed");
    } else {
        LOGI("Main", "I2C bus recovered");
    }
    return boot_i2c_recovered;
}
```

- `recoverI2CBus()` 是标准 I2C 总线恢复程序：将 SCL 配为开漏输出，产生最多 9 个时钟脉冲，如果 SDA 在第 9 个脉冲后仍为低则产生一个 STOP 条件。这能释放被从设备锁死的总线——在多传感器系统中，某个传感器在写入过程中复位会导致它持续拉低 SDA，使整条总线瘫痪。

### 6.4 管理器初始化与配置加载

`initManagersAndConfig()` 展示了各子系统的初始化顺序和依赖关系：

```cpp
void AppInit::initManagersAndConfig(Context &ctx, StorageManager::BootAction boot_action) {
    ctx.storage.begin(boot_action);
    ctx.displayThresholds.begin(ctx.storage);
    ctx.networkManager.begin(ctx.storage);
    ctx.mqttManager.begin(ctx.storage, ctx.networkManager, ctx.mqttRuntimeState);

    g_mqtt_manager = &ctx.mqttManager;
    ctx.networkManager.attachMqttContext(
        ctx.mqttManager,
        ctx.mqttManager.userEnabledRef(),
        ctx.mqttManager.hostRef(),
        ctx.mqttManager.portRef(),
        ctx.mqttManager.userRef(),
        ctx.mqttManager.passRef(),
        ctx.mqttManager.deviceNameRef(),
        ctx.mqttManager.baseTopicRef(),
        ctx.mqttManager.deviceIdRef(),
        ctx.mqttManager.discoveryRef(),
        ctx.mqttManager.anonymousRef(),
        mqtt_sync_with_wifi_cb);
    ctx.networkManager.attachThemeContext(ctx.themeManager);
    // ... 更多 attach 调用 ...

    const auto &cfg = ctx.storage.config();
    UiStrings::setLanguage(cfg.language);
    ctx.temp_offset = cfg.temp_offset;
    ctx.hum_offset = cfg.hum_offset;
    InitConfig::normalizeOffsets(ctx.temp_offset, ctx.hum_offset);
    ctx.temp_units_c = cfg.units_c;
    ctx.night_mode = cfg.night_mode;
    ctx.co2_asc_enabled = cfg.asc_enabled;
    ctx.themeManager.loadFromPrefs(ctx.storage);
```

**逐段解析**：

- 初始化严格按依赖顺序：`storage` 最先（其他管理器需要从存储加载配置），然后 `networkManager` 和 `mqttManager`（MQTT 依赖网络）。
- `attachMqttContext()` 将 MQTT 管理器的各种配置引用（`userEnabledRef()`、`hostRef()` 等）注入网络管理器。这种**引用注入**模式让网络管理器能直接读写 MQTT 配置，而不需要每次通过函数调用中转——在 ESP32 的并发场景下减少了锁竞争。
- `mqtt_sync_with_wifi_cb` 是回调函数：当 Wi-Fi 状态变化时通知 MQTT 管理器同步连接状态。这让两个子系统保持松耦合——网络管理器不知道 MQTT 的内部逻辑，只通过回调通知"Wi-Fi 变了"。
- 从 `storage.config()` 加载用户偏好：语言、温度单位（摄氏/华氏）、夜间模式、温湿度偏移校准、CO2 ASC（自动基线校准）等。`normalizeOffsets()` 钳制偏移值到合理范围。

### 6.5 SEN66 传感器 I2C 驱动（Sen66.cpp）

SEN66 是 Sensirion 的多合一环境传感器，I2C 通信带 CRC-8 校验。驱动封装了寄存器读写：

```cpp
bool Sen66::writeCmdWithWord(uint16_t cmd, uint16_t word) {
    uint8_t params[3] = {
        static_cast<uint8_t>(word >> 8),
        static_cast<uint8_t>(word & 0xFF),
        0
    };
    params[2] = I2C::crc8(params, 2);
    return I2C::write_cmd(Config::SEN66_ADDR, cmd, params, sizeof(params)) == ESP_OK;
}

bool Sen66::writeCmdWithWords(uint16_t cmd, const uint16_t *words, size_t count) {
    if (!words || count == 0 || count > 8) {
        return false;
    }
    uint8_t params[8 * 3] = {};
    for (size_t i = 0; i < count; ++i) {
        const size_t off = i * 3;
        params[off] = static_cast<uint8_t>(words[i] >> 8);
        params[off + 1] = static_cast<uint8_t>(words[i] & 0xFF);
        params[off + 2] = I2C::crc8(&params[off], 2);
    }
    return I2C::write_cmd(Config::SEN66_ADDR, cmd, params, count * 3) == ESP_OK;
}
```

**逐段解析**：

- Sensirion 传感器的 I2C 协议规定：每个 16 位数据字后跟一个 **CRC-8** 校验字节（多项式 0x31，初始值 0xFF）。`writeCmdWithWord()` 将 16 位命令参数拆为 2 字节 + 1 字节 CRC，共 3 字节负载。
- `writeCmdWithWords()` 处理多字参数：每 2 字节一组附 CRC，最多 8 组（24 字节）。`8 * 3` 的栈缓冲区在编译期确定大小，避免动态分配。
- `I2C::crc8()` 是 Sensirion 标准的 CRC-8 计算，对每 2 字节计算校验。这种**每字校验**设计能在传输错误时精确定位损坏的字，而非整包丢弃。

温度偏移校准是一个重要的传感器特性：

```cpp
bool Sen66::setTemperatureOffsetParams(float offset_c, float slope, uint16_t time_constant_s, uint16_t slot) {
    const int16_t offset_scaled = static_cast<int16_t>(lroundf(offset_c * 200.0f));
    const int16_t slope_scaled = static_cast<int16_t>(lroundf(slope * 10000.0f));
    const uint16_t words[4] = {
        static_cast<uint16_t>(offset_scaled),
        static_cast<uint16_t>(slope_scaled),
        time_constant_s,
        slot
    };
    if (!writeCmdWithWords(Config::SEN66_CMD_TEMP_OFFSET, words, 4)) {
        return false;
    }
    delay(Config::SEN66_CMD_DELAY_MS);
    return true;
}
```

**逐段解析**：

- 温度偏移支持**斜率校准**：`offset_c` 是基础偏移（°C），`slope` 是温度斜率（°C/°C），`time_constant_s` 是滤波时间常数。这比简单的固定偏移更精确——传感器自热效应与温度相关，斜率校准能补偿这种非线性。
- `offset_c * 200.0f`：SEN66 的温度偏移分辨率为 0.005°C（即 1/200），所以乘 200 转为整数。`lroundf` 做四舍五入而非截断，减少系统偏差。
- `slope * 10000.0f`：斜率分辨率 0.0001，乘 10000 转整数。
- 4 个 16 位字打包后通过 `writeCmdWithWords` 一次性写入，减少 I2C 事务次数。

数据读取与 CRC 校验：

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
```

**逐段解析**：

- Sensirion 读取流程：先写命令（无参数），等待传感器处理（`delay_ms`），再读取数据。`delay_ms` 因命令而异——测量数据就绪查询只需几毫秒，但某些命令需要更长时间。
- `buf[27]` 对应最多 9 个字（9×3=27 字节），覆盖 SEN66 最大读取长度。栈分配避免堆碎片。
- 逐字 CRC 校验：任何一字校验失败立即返回 `false`，调用方会触发重试机制。`out[i] = (p[0] << 8) | p[1]` 是大端字节序转 16 位整数。

VOC 状态持久化展示了传感器的长期学习特性：

```cpp
void Sen66::saveVocState(StorageManager &storage) {
    if (!ok_ || busy_ || !measuring_) {
        return;
    }
    uint32_t now = millis();
    if (now - last_voc_state_save_ms_ < Config::SEN66_VOC_STATE_SAVE_MS) {
        return;
    }
    uint8_t state[Config::SEN66_VOC_STATE_LEN] = {};
    if (!getVocState(state, sizeof(state))) {
        LOGW("SEN66", "VOC state read failed");
        last_voc_state_save_ms_ = now;
        return;
    }
    memcpy(voc_state_, state, sizeof(voc_state_));
    voc_state_valid_ = true;
    storage.saveVocState(voc_state_, sizeof(voc_state_));
    last_voc_state_save_ms_ = now;
}
```

**逐段解析**：

- SEN66 的 VOC/NOx 算法是**自适应学习型**——传感器内部维护一个基线状态，随使用时间不断学习环境的典型挥发性有机物水平。这个状态如果丢失，传感器需要重新学习（数小时到数天），VOC 读数在此期间不准确。
- 因此固件定期将 VOC 状态保存到 LittleFS（`SEN66_VOC_STATE_SAVE_MS` 间隔），启动时恢复（`loadVocState`）。这是**跨电源周期保持传感器精度**的关键设计。
- `last_voc_state_save_ms_` 节流避免频繁写 Flash（Flash 写入有寿命限制，典型 10 万次）。

### 6.6 GP8403 DAC 驱动（Gp8403.cpp）

GP8403 是双通道 12 位 I2C DAC，输出 0-10V 模拟电压用于驱动风扇/执行器：

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

**逐段解析**：

- `writeChannelMillivolts()` 将毫伏值转换为 12 位原始码值。公式 `raw12 = (millivolts × 4095 + FS/2) / FS` 中，`+ FS/2` 是**四舍五入**而非截断——减少系统性的 -0.5 LSB 偏差。用 `uint32_t` 中间运算避免 16 位溢出（10000 × 4095 = 40,950,000 超出 uint16_t 范围）。
- 输入钳制在 `DAC_VOUT_MIN_MV` 到 `DAC_VOUT_FULL_SCALE_MV` 之间，防止越界。
- `raw12 << 4`：GP8403 的寄存器格式是 16 位对齐——12 位有效数据左移 4 位放在高 12 位，低 4 位为 0。这是 I2C DAC 的常见寄存器布局。
- `payload` 按小端序排列（低字节在前），与 ESP32 的原生字节序和 GP8403 寄存器格式一致。

底层 I2C 写入：

```cpp
bool Gp8403::writeRegister(uint8_t reg, const uint8_t *data, size_t len) {
    if (address_ == 0) {
        return false;
    }
    if (len > 2) {
        return false;
    }

    uint8_t tx[3] = {reg, 0, 0};
    for (size_t i = 0; i < len; ++i) {
        tx[1 + i] = data[i];
    }

    const esp_err_t err = i2c_master_write_to_device(
        Config::I2C_PORT,
        address_,
        tx,
        len + 1,
        pdMS_TO_TICKS(Config::I2C_TIMEOUT_MS)
    );
    return err == ESP_OK;
}
```

**逐段解析**：

- `tx[3] = {reg, 0, 0}` 将寄存器地址和数据拼成一个连续缓冲区——`i2c_master_write_to_device` 需要单次事务写入所有字节（寄存器地址 + 数据）。`len + 1` 包含寄存器地址字节。
- `pdMS_TO_TICKS(Config::I2C_TIMEOUT_MS)` 将毫秒超时转为 FreeRTOS tick 数。这是 ESP-IDF I2C 驱动的标准超时参数，防止总线挂起时永久阻塞。
- 所有错误都返回 `bool`，调用方（FanControl）会根据返回值重试或标记故障。

### 6.7 风扇控制与线程安全（FanControl.cpp）

FanControl 管理风扇的多种运行模式，使用 FreeRTOS 互斥量保证线程安全：

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

void FanControl::unlockSync() const {
    if (sync_mutex_ != nullptr) {
        xSemaphoreGive(sync_mutex_);
    }
}
```

**逐段解析**：

- `ensureSyncPrimitives()` 使用**惰性初始化**——互斥量在首次需要时创建，避免静态初始化阶段 FreeRTOS 尚未就绪的问题。
- `lockSync()` 用 `portMAX_DELAY` 无限等待获取互斥量。这在 FanControl 场景下是安全的——风扇控制不是实时关键路径，短暂阻塞不会影响音频级别的时序。但注意 `sync_mutex_ == nullptr` 时直接返回 `true`（无锁），这是惰性初始化未触发时的回退路径。
- 所有状态访问都通过 `lockSync()`/`unlockSync()` 包裹，因为 FanControl 同时被主循环（Core 0）和网络任务（可能不同核心）访问。

快照发布模式避免长时间持锁：

```cpp
void FanControl::publishSnapshot() {
    if (!lockSync()) {
        return;
    }
    snapshot_.present = present_;
    snapshot_.available = available_;
    snapshot_.running = running_;
    snapshot_.faulted = faulted_;
    snapshot_.output_known = output_known_;
    snapshot_.manual_override_active = manual_override_active_;
    snapshot_.auto_resume_blocked = auto_resume_blocked_;
    snapshot_.mode = mode_;
    snapshot_.manual_step = manual_step_;
    snapshot_.selected_timer_s = selected_timer_s_;
    snapshot_.output_mv = output_mv_;
    snapshot_.stop_at_ms = stop_at_ms_;
    snapshot_.auto_config = auto_config_;
    unlockSync();
}
```

**逐段解析**：

- **快照模式**（Snapshot）是多线程状态共享的经典模式：写线程在锁内将所有状态字段复制到 `snapshot_` 结构体，读线程直接读取 `snapshot_`（不加锁或仅短暂加锁）。这样读取方不会长时间阻塞写方，且看到的是一致的状态视图（所有字段来自同一时刻）。
- `snapshot_` 包含风扇的全部状态：是否在线、是否运行、是否故障、当前模式（手动/定时/自动）、输出电压、停止时间等。MQTT 和 Web 仪表盘都从这个快照读取数据。

### 6.8 构建配置（platformio.ini）

```ini
[env:project_aura]
platform = https://github.com/pioarduino/platform-espressif32/releases/download/53.03.11/platform-espressif32.zip
platform_packages =
    platformio/framework-arduinoespressif32@https://github.com/espressif/arduino-esp32.git#3.1.1
    platformio/framework-arduinoespressif32-libs@https://dl.espressif.com/AE/esp-arduino-libs/esp32-3.1.1-h.zip
board = esp32-s3-devkitc-1
framework = arduino
monitor_speed = 115200

; --- MEMORY (Flash) ---
board_upload.flash_size = 16MB
board_build.partitions = partitions_16MB_littlefs.csv
board_build.filesystem = littlefs

board_build.arduino.memory_type = qio_opi
board_build.sdkconfig = sdkconfig.defaults

build_flags =
    -DCORE_DEBUG_LEVEL=1
    -DAPP_VERSION=\"1.1.5\"
    -DLV_CONF_INCLUDE_SIMPLE
    -DLV_LVGL_H_INCLUDE_SIMPLE
    -DLV_COLOR_16_SWAP=0
    -I include

lib_deps =
    https://github.com/esp-arduino-libs/ESP32_Display_Panel.git
    https://github.com/esp-arduino-libs/ESP32_IO_Expander.git#v1.1.0
    https://github.com/esp-arduino-libs/esp-lib-utils.git#v0.2.0
    https://github.com/lvgl/lvgl.git#v8.4.0
    bblanchon/ArduinoJson@^7.0.0
```

**逐段解析**：

- `framework-arduinoespressif32@...#3.1.1` 锁定 Arduino-ESP32 核心 3.1.1 版本（基于 ESP-IDF 5.3.x）。精确版本锁定保证可重现构建——ESP32 生态变化快，新版核心可能引入 API 破坏。
- `board_upload.flash_size = 16MB` + `partitions_16MB_littlefs.csv`：使用 16MB Flash 的自定义分区表，划分出足够的 LittleFS 空间存放 Web 仪表盘资产（HTML/CSS/JS）、多语言翻译、配置文件和历史数据。
- `board_build.arduino.memory_type = qio_opi`：Flash 使用 QIO（四线 IO）+ OPI PSRAM（八线 PSRAM）。QIO 比 default 的 DIO 快一倍，对 LVGL 界面流畅度影响显著；OPI PSRAM 为 800×480 双缓冲帧缓冲提供足够空间。
- `LV_COLOR_16_SWAP=0`：RGB565 字节序不交换。Waveshare ESP32-S3-Touch-LCD-4.3 的显示驱动期望标准字节序，此设置与硬件匹配。
- 构建脚本 `scripts/set_build_id.py`、`generate_dashboard_gzip.py` 等在编译前预处理：生成唯一构建 ID、将 Web 仪表盘 HTML/CSS 压缩为 gzip 嵌入固件（减少 Flash 占用）。
- 测试环境 `native_test` 使用 PlatformIO 的 native 平台 + Unity 测试框架，在 PC 上运行单元测试——`test_build_src = true` 把源文件编译进测试，配合 `prepend_mocks.py` 注入硬件 mock，实现无硬件的 CI 测试。

---

## 七、API 使用指南

### 7.1 Web API 路由

设备在 STA 模式下提供 REST API（默认端口 80）：

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/state` | 当前所有传感器读数与状态 |
| GET | `/api/charts?group=core\|gases\|pm&window=1h\|3h\|24h` | 历史图表数据 |
| GET | `/api/events` | 事件日志 |
| GET | `/api/diag` | 诊断信息（仅 AP 模式） |
| POST | `/api/settings` | 更新设备设置 |
| POST | `/api/ota` | 上传固件 OTA 升级 |

### 7.2 cURL 调用示例

```bash
# 获取当前传感器状态
curl http://aura-1a2b3c.local/api/state

# 获取过去 24 小时的颗粒物图表数据
curl "http://aura-1a2b3c.local/api/charts?group=pm&window=24h"

# 获取事件日志
curl http://aura-1a2b3c.local/api/events

# 更新设置（JSON）
curl -X POST http://aura-1a2b3c.local/api/settings \
  -H "Content-Type: application/json" \
  -d '{"night_mode": true, "units_c": true}'

# OTA 固件升级
curl -X POST http://aura-1a2b3c.local/api/ota \
  -F "firmware=@firmware.bin"
```

### 7.3 MQTT 主题结构

| 主题 | 方向 | 说明 |
|---|---|---|
| `<base>/state` | 设备→HA | 所有传感器状态 JSON |
| `<base>/status` | 设备→HA | 在线/离线（LWT 遗嘱） |
| `<base>/command/*` | HA→设备 | 命令（night_mode, backlight, restart 等） |
| `homeassistant/*/config` | 设备→HA | Home Assistant 自动发现配置 |

```bash
# 订阅设备状态（mosquitto 客户端）
mosquitto_sub -h <mqtt_broker> -t "aura/+/state" -v

# 发送夜间模式命令
mosquitto_pub -h <mqtt_broker> -t "aura/aura-1a2b3c/command/night_mode" -m "ON"
```

---

## 八、编译与部署

### 8.1 环境搭建

| 组件 | 版本要求 |
|---|---|
| 构建系统 | PlatformIO CLI 或 VSCode + PlatformIO 扩展 |
| Arduino-ESP32 核心 | 3.1.1（ESP-IDF 5.3.x） |
| Python | 3.x（构建脚本依赖） |
| 目标板 | ESP32-S3-Touch-LCD-4.3（16MB Flash） |

### 8.2 编译配置表

| 配置项 | 值 | 说明 |
|---|---|---|
| Flash 大小 | 16MB | QIO 模式 |
| 分区表 | partitions_16MB_littlefs.csv | 含 LittleFS 数据分区 |
| PSRAM | OPI | 八线 PSRAM |
| 调试级别 | 1 (ERROR) | 减少日志开销 |
| LVGL 版本 | 8.4.0 | 锁定版本 |
| 显示面板库 | ESP32_Display_Panel | Waveshare 板支持 |

### 8.3 编译与烧录步骤

```bash
git clone https://github.com/21cncstudio/project_aura.git
cd project_aura

# 复制密钥模板（可选，用于预置 Wi-Fi/MQTT 凭证）
cp include/secrets.h.example include/secrets.h
# 编辑 secrets.h 填入你的配置

# 编译固件
pio run -e project_aura

# 烧录固件
pio run -e project_aura -t upload

# 烧录 LittleFS 文件系统镜像（Web/UI 资产，首次必须执行）
pio run -e project_aura -t uploadfs

# 串口监视器
pio device monitor -b 115200
```

### 8.4 使用方法

1. **首次开机**：设备创建热点 `Aura-XXXXXX-AP`，连接后访问 `http://192.168.4.1` 配置 Wi-Fi
2. **配网后**：等待约 15 秒 AP→STA 切换，访问 `http://aura-XXXXXX.local/dashboard`
3. **Home Assistant**：MQTT 发现已默认启用，在 HA 的 MQTT 集成中自动出现
4. **OTA 升级**：Web 仪表盘 → System → 上传 `.bin` 固件文件
5. **传感器预热**：SEN66 的 VOC/NOx 需约 5 分钟预热，UI 显示 WARMUP 状态

---

## 九、项目亮点与适用场景

### 技术亮点

- **安全启动回滚机制**：崩溃自动计数 + 配置回滚 + 工厂复位三级保护，杜绝 OTA 变砖
- **I2C 总线恢复**：启动时自动执行 9 时钟脉冲恢复，解决多传感器总线锁死
- **线程安全快照模式**：FanControl 用互斥量 + 快照复制，避免长时间持锁
- **VOC 状态持久化**：跨电源周期保持传感器自适应学习状态，避免重新校准
- **独立网络任务**：Wi-Fi/MQTT/Web 在专用任务运行，不阻塞传感器主循环
- **OTA 期间 LVGL 协调**：固件升级时暂停显示刷新，避免 Flash Cache 冲突 panic
- **无硬件单元测试**：native 平台 + mock 注入实现 CI 全自动化测试
- **硬件自探测**：多款 RTC/气压传感器自动识别，即插即用

### 适用场景

- 室内空气质量监测（家庭/办公室/教室）
- 智能家居环境感知节点（Home Assistant 集成）
- 新风/净化系统联动控制（0-10V DAC 输出）
- CO2 浓度监测与通风提醒
- DIY 桌面气象站（温湿度/气压/多气体）
- 甲醛/有害气体长期监测
- 嵌入式 IoT 全栈学习参考（传感器驱动/Web/MQTT/LVGL/OTA）

---

## 十、总结

Project Aura 是一个完成度极高的 ESP32-S3 空气质量监测站项目。它没有停留在"读传感器串口打印"的 Demo 层面，而是构建了一套从硬件驱动、安全启动、网络通信、图形界面到智能家居集成的**完整产品级固件**。

我最欣赏的是它在可靠性上的工程投入：安全启动回滚机制解决了 OTA 变砖痛点，I2C 总线恢复应对多传感器总线锁死，VOC 状态持久化保持传感器长期精度，OTA 期间的 LVGL 协调避免了 Flash Cache panic——这些都是量产产品才会考虑的细节。代码组织清晰，`core/modules/drivers/ui/web` 的分层让每个模块职责单一，配合 native 平台的单元测试，维护性很好。对于想学习 ESP32 全栈开发（传感器 I2C 驱动、LVGL GUI、Web 服务器、MQTT 智能家居、OTA 升级、安全启动）的开发者，Project Aura 是一份非常值得研读的参考实现。

---

> 📝 作者：蔡浩宇（jun-chy）
>
> 📅 日期：2026-06-28
>
> 🔗 项目地址：https://github.com/21cncstudio/project_aura
