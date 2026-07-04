# ESP32 电子纸天气显示器深度解析 — esp32-weather-epd

> 📅 日期：2026-07-04 | 🏷️ 分类：ESP32 | ⭐ Stars：6,165 | 👤 作者：蔡浩宇（jun-chy）

---

## 项目信息

| 属性 | 详情 |
|------|------|
| **GitHub 地址** | [lmarzen/esp32-weather-epd](https://github.com/lmarzen/esp32-weather-epd) |
| **作者** | Luke Marzen (lmarzen) |
| **许可证** | GPL-3.0 |
| **最近更新** | 2026-07-02 |
| **主要语言** | C++ |
| **Forks** | 437+ |
| **默认分支** | main |

---

## 一、项目简介

esp32-weather-epd 是一个基于 ESP32 微控制器和 7.5 寸电子纸显示屏的超低功耗天气显示器项目。它通过 WiFi 从 OpenWeatherMap API 获取天气数据，同时利用板载 BME280 传感器读取室内温湿度，最终在电子纸上渲染出精美的天气信息面板。

该项目的核心亮点在于其极致的低功耗设计：休眠时功耗仅约 14μA，刷新时约 83mA（持续约 15 秒），一颗 5000mAh 电池可运行 6-12 个月。

### 核心特性一览

| 特性 | 说明 |
|------|------|
| 超低功耗 | 休眠 ~14μA，刷新 ~83mA/15s |
| 长续航 | 5000mAh 电池可运行 6-12 个月 |
| 多面板支持 | 黑白/红黑白/七色电子纸，800×480px |
| 多语言 | 支持 11 种语言区域设置 |
| 多单位 | 温度/风速/气压/能见度/降水均可切换 |
| 电池监控 | 四级电池电压保护策略 |
| 智能休眠 | 对齐分钟边界 + 夜间省电模式 |
| 室内传感器 | BME280/BME680 可选 |
| HTTPS 安全通信 | 支持 X.509 证书验证 |
| 模块化字体 | 14 种开源字体可切换 |

---

## 二、硬件架构

### 系统组成图

```
                    ┌──────────────────────────────────┐
                    │         OpenWeatherMap API        │
                    │    (One Call 3.0 + Air Pollution) │
                    └──────────────┬───────────────────┘
                                   │ HTTPS over WiFi
                    ┌──────────────▼───────────────────┐
                    │       ESP32 (FireBeetle 2 E)      │
                    │  ┌─────────────────────────────┐  │
                    │  │  WiFi  │  RTC  │  ADC  │GPIO│  │
                    │  └────┬───┴───┬───┴───┬───┴────┘  │
                    │       │       │       │           │
                    └───────┼───────┼───────┼───────────┘
                            │       │       │
              ┌─────────────┼───┐   │   ┌───┼────────────┐
              │             │   │   │   │   │            │
        ┌─────▼─────┐ ┌────▼───▼┐ │ ┌─▼───▼────┐ ┌──────▼──────┐
        │  DESPI-C02 │ │  SPI    │ │ │  I2C     │ │  LiPo Battery│
        │  Driver    │ │  Bus    │ │ │  Bus     │ │  3.7V JST-PH │
        │  Board     │ │         │ │ │          │ │  (5000mAh)   │
        └─────┬──────┘ └─────────┘ │ └──────────┘ └─────────────┘
              │                    │
        ┌─────▼──────┐      ┌──────▼──────┐
        │ 7.5" E-Paper│      │   BME280    │
        │  800×480px  │      │  温湿度气压  │
        │  (B/W/3C/7C)│      │   传感器     │
        └────────────┘      └─────────────┘
```

### 引脚配置

以下引脚定义来自项目源码 `config.cpp`，针对 FireBeetle 2 ESP32-E 开发板：

| 功能 | 引脚 | 说明 |
|------|------|------|
| 电池电压 ADC | A2 | 测量锂电池电压 |
| EPD BUSY | GPIO 14 | 电子纸忙信号 |
| EPD CS | GPIO 13 | SPI 片选 |
| EPD RST | GPIO 21 | 电子纸复位 |
| EPD DC | GPIO 22 | 数据/命令选择 |
| EPD SCK | GPIO 18 | SPI 时钟 |
| EPD MISO | GPIO 19 | 主入从出（未使用） |
| EPD MOSI | GPIO 23 | 主出从入 |
| EPD 电源 | GPIO 26 | 电子纸供电控制 |
| BME SDA | GPIO 17 | I2C 数据 |
| BME SCL | GPIO 16 | I2C 时钟 |
| BME 电源 | GPIO 4 | 传感器供电控制 |

### 硬件清单

| 组件 | 型号 | 说明 |
|------|------|------|
| MCU | FireBeetle 2 ESP32-E | 低功耗设计，USB-C，电池管理 |
| 显示屏 | Waveshare 7.5" E-Paper v2 | 800×480px，黑/白 |
| 驱动板 | DESPI-C02 | 推荐使用，RESE 设为 0.47 |
| 传感器 | BME280 | 温度/湿度/气压，I2C 0x76 |
| 电池 | 3.7V LiPo (JST-PH2.0) | 5000mAh 可用 6+ 个月 |
| 充电 | USB-C | 通过 FireBeetle 板载充电 |

---

## 三、固件架构

### 文件结构

```
platformio/
├── platformio.ini          # PlatformIO 构建配置
├── src/
│   ├── main.cpp            # 主程序入口（setup/loop）
│   ├── config.cpp          # 引脚/WiFi/API/时间/电池配置
│   ├── client_utils.cpp    # HTTP 客户端工具（WiFi/时间同步）
│   ├── renderer.cpp        # 显示渲染引擎
│   ├── display_utils.cpp   # 显示辅助函数
│   ├── owm_api.cpp         # OpenWeatherMap API 请求
│   ├── api_response.h      # API 响应数据结构
│   └── locales/            # 多语言支持（11种）
├── include/
│   ├── config.h            # 编译时配置（面板/传感器/单位）
│   └── _locale.h           # 语言区域接口
└── lib/
    ├── icons/              # 天气图标资源
    └── fonts/              # 14 种开源字体
```

### 模块职责

| 模块 | 文件 | 职责 |
|------|------|------|
| 主控逻辑 | `main.cpp` | 状态机编排：电池检查→WiFi→NTP→API→渲染→休眠 |
| 硬件配置 | `config.cpp` | 引脚映射、WiFi 凭证、API Key、休眠周期 |
| 编译配置 | `config.h` | 面板型号、传感器型号、单位制、语言区域 |
| 网络通信 | `client_utils.cpp` | WiFi 连接/断开、SNTP 时间同步 |
| API 交互 | `owm_api.cpp` | OpenWeatherMap One Call + Air Pollution 请求 |
| 渲染引擎 | `renderer.cpp` | 天气面板布局绘制（当前天气/预报/图表/状态栏） |
| 多语言 | `locales/` | 11 种语言的文本本地化 |

---

## 四、核心代码深度分析

### 4.1 主程序状态机（main.cpp）

整个固件的核心是 `setup()` 函数中的状态机。ESP32 每次从深度睡眠唤醒后都从 `setup()` 重新开始执行，`loop()` 永远不会运行。以下是主程序的关键流程：

```cpp
void setup()
{
  unsigned long startTime = millis();
  Serial.begin(115200);
#if DEBUG_LEVEL >= 1
  printHeapUsage();
#endif
  disableBuiltinLED();
  // Open namespace for read/write to non-volatile storage
  prefs.begin(NVS_NAMESPACE, false);
```

**逐行解析：**
- `startTime = millis()`：记录唤醒时间戳，用于后续计算本次唤醒总耗时
- `disableBuiltinLED()`：关闭板载 LED 以降低功耗， FireBeetle 的 LED 在深度睡眠时仍可能消耗电流
- `prefs.begin(NVS_NAMESPACE, false)`：打开非易失性存储（NVS）命名空间，`false` 表示读写模式，用于持久化低电量标志位

### 4.2 四级电池保护策略

这是该项目最精妙的设计之一。通过四级电压阈值实现电池的渐进式保护：

```cpp
#if BATTERY_MONITORING
  uint32_t batteryVoltage = readBatteryVoltage();
  Serial.print(TXT_BATTERY_VOLTAGE);
  Serial.println(": " + String(batteryVoltage) + "mv");
  bool lowBat = prefs.getBool("lowBat", false);
  // low battery, deep sleep now
  if (batteryVoltage <= LOW_BATTERY_VOLTAGE)
  {
    if (lowBat == false)
    { // battery is now low for the first time
      prefs.putBool("lowBat", true);
      prefs.end();
      initDisplay();
      do
      {
        drawError(battery_alert_0deg_196x196, TXT_LOW_BATTERY);
      } while (display.nextPage());
      powerOffDisplay();
    }
    if (batteryVoltage <= CRIT_LOW_BATTERY_VOLTAGE)
    { // critically low battery
      // We won't wake up again until someone manually presses the RST button.
      Serial.println(TXT_CRIT_LOW_BATTERY_VOLTAGE);
      Serial.println(TXT_HIBERNATING_INDEFINITELY_NOTICE);
    }
    else if (batteryVoltage <= VERY_LOW_BATTERY_VOLTAGE)
    { // very low battery
      esp_sleep_enable_timer_wakeup(VERY_LOW_BATTERY_SLEEP_INTERVAL
                                    * 60ULL * 1000000ULL);
    }
    else
    { // low battery
      esp_sleep_enable_timer_wakeup(LOW_BATTERY_SLEEP_INTERVAL
                                    * 60ULL * 1000000ULL);
    }
    esp_deep_sleep_start();
  }
```

**逐段解析：**

1. **`readBatteryVoltage()`**：通过 ADC（A2 引脚）读取电池电压，返回毫伏值
2. **`prefs.getBool("lowBat", false)`**：从 NVS 读取上次的低电量标志。这个标志确保低电量警告画面只显示一次——首次检测到低电量时显示警告，之后直接进入休眠，直到电压恢复
3. **四级保护逻辑：**
   - **正常（> 3462mV / ~10%）**：继续正常工作流程
   - **低电量（3462mV / ~10%）**：首次显示低电量图标，然后每 30 分钟唤醒检查一次
   - **极低（3442mV / ~8%）**：每 120 分钟唤醒检查一次
   - **临界（3404mV / ~5%）**：进入无限休眠（hibernate），不设定时器唤醒，只能手动按 RST 按钮恢复

对应的电压阈值在 `config.cpp` 中定义：

```cpp
const uint32_t WARN_BATTERY_VOLTAGE = 3535; // (millivolts) ~20%
const uint32_t LOW_BATTERY_VOLTAGE = 3462; // (millivolts) ~10%
const uint32_t VERY_LOW_BATTERY_VOLTAGE = 3442; // (millivolts) ~8%
const uint32_t CRIT_LOW_BATTERY_VOLTAGE = 3404; // (millivolts) ~5%
const unsigned long LOW_BATTERY_SLEEP_INTERVAL = 30; // (minutes)
const unsigned long VERY_LOW_BATTERY_SLEEP_INTERVAL = 120; // (minutes)
```

### 4.3 WiFi 连接与错误处理

```cpp
  int wifiRSSI = 0;
  wl_status_t wifiStatus = startWiFi(wifiRSSI);
  if (wifiStatus != WL_CONNECTED)
  { // WiFi Connection Failed
    killWiFi();
    initDisplay();
    if (wifiStatus == WL_NO_SSID_AVAIL)
    {
      do
      {
        drawError(wifi_x_196x196, TXT_NETWORK_NOT_AVAILABLE);
      } while (display.nextPage());
    }
    else
    {
      do
      {
        drawError(wifi_x_196x196, TXT_WIFI_CONNECTION_FAILED);
      } while (display.nextPage());
    }
    powerOffDisplay();
    beginDeepSleep(startTime, &timeInfo);
  }
```

**逐行解析：**
- `startWiFi(wifiRSSI)`：连接 WiFi 并通过引用返回信号强度（RSSI），函数内部有 10 秒超时
- `killWiFi()`：立即关闭 WiFi 模块以省电——这是低功耗设计的关键，WiFi 只在需要获取数据时开启
- `initDisplay()` → `drawError()` → `powerOffDisplay()`：电子纸的错误显示流程。`do...while (display.nextPage())` 是 GxEPD2 库的分页渲染模式，电子纸显存较大，需要分页写入
- `beginDeepSleep()`：无论成功失败，最终都进入深度睡眠等待下次唤醒

### 4.4 深度睡眠对齐算法

这是整个项目最精妙的代码段。它不是简单地休眠固定时长，而是将唤醒时间对齐到分钟边界：

```cpp
void beginDeepSleep(unsigned long startTime, tm *timeInfo)
{
  if (!getLocalTime(timeInfo))
  {
    Serial.println(TXT_REFERENCING_OLDER_TIME_NOTICE);
  }
  int bedtimeHour = INT_MAX;
  if (BED_TIME != WAKE_TIME)
  {
    bedtimeHour = (BED_TIME - WAKE_TIME + 24) % 24;
  }
  // time is relative to wake time
  int curHour = (timeInfo->tm_hour - WAKE_TIME + 24) % 24;
  const int curMinute = curHour * 60 + timeInfo->tm_min;
  const int curSecond = curHour * 3600
                      + timeInfo->tm_min * 60
                      + timeInfo->tm_sec;
  const int desiredSleepSeconds = SLEEP_DURATION * 60;
  const int offsetMinutes = curMinute % SLEEP_DURATION;
  const int offsetSeconds = curSecond % desiredSleepSeconds;
  // align wake time to nearest multiple of SLEEP_DURATION
  int sleepMinutes = SLEEP_DURATION - offsetMinutes;
  if (desiredSleepSeconds - offsetSeconds < 120
   || offsetSeconds / (float)desiredSleepSeconds > 0.95f)
  { // if we have a sleep time less than 2 minutes OR less 5% SLEEP_DURATION,
    // skip to next alignment
    sleepMinutes += SLEEP_DURATION;
  }
  const int predictedWakeHour = ((curMinute + sleepMinutes) / 60) % 24;
  uint64_t sleepDuration;
  if (predictedWakeHour < bedtimeHour)
  {
    sleepDuration = sleepMinutes * 60 - timeInfo->tm_sec;
  }
  else
  {
    const int hoursUntilWake = 24 - curHour;
    sleepDuration = hoursUntilWake * 3600ULL
                    - (timeInfo->tm_min * 60ULL + timeInfo->tm_sec);
  }
  sleepDuration += 3ULL;
  sleepDuration *= 1.0015f;
  esp_sleep_enable_timer_wakeup(sleepDuration * 1000000ULL);
  esp_deep_sleep_start();
}
```

**逐段深度解析：**

1. **时间基准转换**：`(BED_TIME - WAKE_TIME + 24) % 24` 将时间转换为相对于 WAKE_TIME 的偏移量。例如 WAKE_TIME=6, BED_TIME=0，则 bedtimeHour=18，表示从 6:00 算起 18 小时后是就寝时间

2. **分钟对齐**：`offsetMinutes = curMinute % SLEEP_DURATION` 计算当前时间相对于 SLEEP_DURATION 周期的偏移。例如 SLEEP_DURATION=30，当前 10:17，则 curMinute=4*60+17=257，offset=257%30=17，需要再睡 30-17=13 分钟到 10:30

3. **短休眠跳过**：如果计算出的剩余睡眠时间不足 2 分钟或不足总周期的 5%，直接跳到下一个对齐点。这避免了频繁唤醒导致的高功耗

4. **夜间省电模式**：`predictedWakeHour >= bedtimeHour` 时，直接休眠到第二天 WAKE_TIME。例如 BED_TIME=0, WAKE_TIME=6，晚上 23:50 唤醒后不会在 00:20 再唤醒，而是直接睡到 06:00

5. **RTC 校准补偿**：`sleepDuration += 3ULL` 和 `sleepDuration *= 1.0015f` 是对 ESP32 RTC 时钟偏差的经验补偿。不同 ESP32 的 RTC 有快有慢，0.15% 的补偿系数是实测得出的平均值

### 4.5 OpenWeatherMap API 请求

```cpp
#ifdef USE_HTTP
  WiFiClient client;
#elif defined(USE_HTTPS_NO_CERT_VERIF)
  WiFiClientSecure client;
  client.setInsecure();
#elif defined(USE_HTTPS_WITH_CERT_VERIF)
  WiFiClientSecure client;
  client.setCACert(cert_Sectigo_Public_Server_Authentication_Root_R46);
#endif
  int rxStatus = getOWMonecall(client, owm_onecall);
  if (rxStatus != HTTP_CODE_OK)
  {
    killWiFi();
    statusStr = "One Call " + OWM_ONECALL_VERSION + " API";
    tmpStr = String(rxStatus, DEC) + ": " + getHttpResponsePhrase(rxStatus);
    initDisplay();
    do
    {
      drawError(wi_cloud_down_196x196, statusStr, tmpStr);
    } while (display.nextPage());
    powerOffDisplay();
    beginDeepSleep(startTime, &timeInfo);
  }
  rxStatus = getOWMairpollution(client, owm_air_pollution);
```

**逐行解析：**
- 三种 HTTP 模式通过编译宏选择：明文 HTTP（最省电）、HTTPS 无证书验证（加密但无认证）、HTTPS 带证书验证（最安全）。默认使用 HTTPS 带证书验证
- `client.setCACert(...)`：硬编码根证书到固件中。注释中提醒用户证书会过期，需要定期用 `cert.py` 脚本更新
- `getOWMonecall()`：请求 One Call 3.0 API，获取当前天气、小时预报、每日预报、警报等完整数据
- `getOWMairpollution()`：请求 Air Pollution API，获取空气质量指数（AQI）
- 两个 API 调用之间不复用连接的检查——如果第一个失败就直接休眠，避免浪费电量

### 4.6 BME280 室内传感器读取

```cpp
  pinMode(PIN_BME_PWR, OUTPUT);
  digitalWrite(PIN_BME_PWR, HIGH);
#if defined(SENSOR_INIT_DELAY_MS) && SENSOR_INIT_DELAY_MS > 0
  delay(SENSOR_INIT_DELAY_MS);
#endif
  TwoWire I2C_bme = TwoWire(0);
  I2C_bme.begin(PIN_BME_SDA, PIN_BME_SCL, 100000); // 100kHz
  float inTemp     = NAN;
  float inHumidity = NAN;
#if defined(SENSOR_BME280)
  Adafruit_BME280 bme;
  if(bme.begin(BME_ADDRESS, &I2C_bme))
  {
    inTemp     = bme.readTemperature(); // Celsius
    inHumidity = bme.readHumidity();    // %
    if (std::isnan(inTemp) || std::isnan(inHumidity))
    {
      statusStr = "BME " + String(TXT_READ_FAILED);
    }
  }
  digitalWrite(PIN_BME_PWR, LOW);
```

**逐行解析：**
- `digitalWrite(PIN_BME_PWR, HIGH)`：通过 GPIO 4 给 BME280 上电。这种 GPIO 控制供电的设计确保传感器在休眠期间完全断电，避免待机功耗
- `delay(SENSOR_INIT_DELAY_MS)`：BME280 上电后需要稳定时间，某些批次需要 300ms
- `TwoWire(0)`：使用 I2C 总线 0（ESP32 有两个 I2C 总线），100kHz 标准模式
- `bme.begin(BME_ADDRESS, &I2C_bme)`：BME_ADDRESS=0x76（SDO 接 GND）或 0x77（SDO 接 VCC）
- `std::isnan()` 检查：如果读取失败返回 NAN，后续渲染时显示 `-` 而非错误数据
- `digitalWrite(PIN_BME_PWR, LOW)`：读取完毕立即断电

### 4.7 显示渲染流程

```cpp
  initDisplay();
  do
  {
    drawCurrentConditions(owm_onecall.current, owm_onecall.daily[0],
                          owm_air_pollution, inTemp, inHumidity);
    drawOutlookGraph(owm_onecall.hourly, owm_onecall.daily, timeInfo);
    drawForecast(owm_onecall.daily, timeInfo);
    drawLocationDate(CITY_STRING, dateStr);
#if DISPLAY_ALERTS
    drawAlerts(owm_onecall.alerts, CITY_STRING, dateStr);
#endif
    drawStatusBar(statusStr, refreshTimeStr, wifiRSSI, batteryVoltage);
  } while (display.nextPage());
  powerOffDisplay();
```

**逐行解析：**
- `initDisplay()`：初始化 GxEPD2 库并唤醒电子纸驱动板
- `do...while (display.nextPage())`：GxEPD2 的分页渲染循环。800×480px 的显存约 48KB，ESP32 的 RAM 足以处理，但库仍使用分页机制以支持更高效的刷新
- **渲染顺序**：
  1. `drawCurrentConditions`：当前天气（大图标+温度+体感+10个信息小格）
  2. `drawOutlookGraph`：24小时温度曲线+降水概率柱状图
  3. `drawForecast`：5天每日预报（图标+高低温）
  4. `drawLocationDate`：城市名+日期（右上角）
  5. `drawAlerts`：天气警报（可选）
  6. `drawStatusBar`：底部状态栏（刷新时间+WiFi信号+电池）
- `powerOffDisplay()`：渲染完成后给电子纸驱动板发送断电指令

### 4.8 编译时配置系统（config.h）

项目使用 C 预处理器宏实现编译时配置，并通过 XOR 校验确保配置合法：

```cpp
// E-PAPER PANEL - 只能选一个
#define DISP_BW_V2
// #define DISP_3C_B
// #define DISP_7C_F
// #define DISP_BW_V1

// INDOOR ENVIRONMENT SENSOR - 只能选一个
#define SENSOR_BME280
// #define SENSOR_BME680

// CONFIG VALIDATION - DO NOT MODIFY
#if !(  defined(DISP_BW_V2)  \
      ^ defined(DISP_3C_B)   \
      ^ defined(DISP_7C_F)   \
      ^ defined(DISP_BW_V1))
  #error Invalid configuration. Exactly one display panel must be selected.
#endif
```

**逐行解析：**
- `#define DISP_BW_V2`：选择 7.5 寸黑白 v2 面板（800×480px），其他选项包括三色（红黑白）、七色 ACeP、旧版 v1（640×384px）
- `^` 运算符（XOR）：用于编译时校验——恰好一个宏被定义时 XOR 结果为 1，否则触发 `#error`。这是一个优雅的编译时互斥校验技巧
- 同样的模式应用于传感器选择、单位制选择等所有配置项

---

## 五、API 使用指南

### cURL 获取天气数据

```bash
# One Call 3.0 API - 获取完整天气数据
curl -s "https://api.openweathermap.org/data/3.0/onecall?lat=40.7128&lon=-74.0060&appid=YOUR_API_KEY&units=metric&exclude=minutely" | python3 -m json.tool

# Air Pollution API - 获取空气质量
curl -s "https://api.openweathermap.org/data/2.5/air_pollution?lat=40.7128&lon=-74.0060&appid=YOUR_API_KEY" | python3 -m json.tool
```

### Python 集成示例

```python
import requests
import json

API_KEY = "your_openweathermap_api_key"
LAT = "40.7128"
LON = "-74.0060"

# 获取 One Call 天气数据
url = f"https://api.openweathermap.org/data/3.0/onecall"
params = {
    "lat": LAT,
    "lon": LON,
    "appid": API_KEY,
    "units": "metric",
    "exclude": "minutely"
}
resp = requests.get(url, params=params, timeout=10)
data = resp.json()

print(f"当前温度: {data['current']['temp']}°C")
print(f"体感温度: {data['current']['feels_like']}°C")
print(f"湿度: {data['current']['humidity']}%")
print(f"天气: {data['current']['weather'][0]['description']}")

# 获取空气质量
air_url = f"https://api.openweathermap.org/data/2.5/air_pollution"
air_params = {"lat": LAT, "lon": LON, "appid": API_KEY}
air_resp = requests.get(air_url, params=air_params, timeout=10)
air_data = air_resp.json()
aqi = air_data['list'][0]['main']['aqi']
aqi_labels = ["优", "良", "轻度污染", "中度污染", "重度污染"]
print(f"空气质量: {aqi_labels[aqi-1]} (AQI={aqi})")
```

---

## 六、编译与部署

### 环境搭建

| 步骤 | 工具 | 说明 |
|------|------|------|
| IDE | VSCode | 推荐 |
| 构建系统 | PlatformIO | VSCode 扩展安装 |
| 框架 | Arduino | ESP32 Arduino Core |
| 依赖 | 自动管理 | PlatformIO 自动下载 |

### 编译配置

```ini
# platformio.ini 核心配置
[env:firebeetle32e]
platform = espressif32
board = firebeetle32e
framework = arduino
monitor_speed = 115200
```

### 烧录步骤

1. **克隆仓库**
   ```bash
   git clone https://github.com/lmarzen/esp32-weather-epd.git
   ```

2. **安装 PlatformIO**
   - VSCode → 扩展 → 搜索 "PlatformIO IDE" → 安装

3. **打开项目**
   - VSCode → File → Open Folder → 选择 `platformio/` 目录

4. **配置参数**
   - 编辑 `src/config.cpp`：设置 WiFi SSID/密码、API Key、经纬度、城市名
   - 编辑 `include/config.h`：选择面板型号、传感器型号、单位制、语言

5. **编译上传**
   - USB 连接 FireBeetle 2 ESP32-E
   - 点击 VSCode 底部的 → 箭头（PlatformIO: Upload）
   - 如遇 `Wrong boot mode detected (0x13)`：断开电源，GPIO0 接 GND，重新上电

6. **OpenWeatherMap API Key**
   - 注册 [openweathermap.org](https://openweathermap.org/api)
   - 订阅 One Call 3.0（每天 1000 次免费调用）
   - 将 API Key 填入 `config.cpp` 的 `OWM_APIKEY`

### 低功耗优化清单

| 优化项 | 方法 | 节省功耗 |
|--------|------|----------|
| 板载 LED | `disableBuiltinLED()` | ~500μA |
| WiFi 模块 | 数据获取后立即 `killWiFi()` | ~80mA |
| BME280 | GPIO 控制供电，读后断电 | ~3.5μA |
| 电子纸驱动板 | 渲染后 `powerOffDisplay()` | ~5mA |
| 深度睡眠 | `esp_deep_sleep_start()` | 降至 ~14μA |
| FireBeetle 焊盘 | 切断低功耗焊盘 | 再降 ~500μA |

---

## 七、项目亮点与适用场景

### 技术亮点

1. **极致低功耗设计**：从硬件选型（FireBeetle 低功耗版）到软件策略（GPIO 控制外设供电、四级电池保护、RTC 校准补偿），全方位优化功耗至 14μA 级别
2. **智能休眠对齐**：不是简单的定时唤醒，而是将唤醒时间对齐到分钟边界，并支持夜间省电模式直接跳过更新
3. **编译时配置系统**：通过预处理器宏 + XOR 校验实现零运行时开销的模块化配置，支持多种面板/传感器/单位组合
4. **渐进式错误处理**：每个环节（WiFi/NTP/API/传感器）都有独立的错误画面和恢复策略，确保系统鲁棒性
5. **HTTPS 证书验证**：支持完整的 X.509 证书链验证，兼顾安全与功耗

### 适用场景

| 场景 | 说明 |
|------|------|
| 家庭天气站 | 壁挂式天气信息中心 |
| 低功耗 IoT 参考 | 超低功耗 ESP32 设计范例 |
| 电子纸应用开发 | GxEPD2 分页渲染学习 |
| OpenWeatherMap 集成 | API 调用与错误处理最佳实践 |
| 电池供电设备 | 四级保护策略可直接复用 |

---

## 八、总结

esp32-weather-epd 是一个在低功耗嵌入式设计领域堪称教科书级别的项目。它不仅仅是"在 ESP32 上显示天气"，而是从硬件选型、功耗管理、错误处理、配置系统到渲染引擎，每一个环节都经过精心设计。其 14μA 休眠功耗和四级电池保护策略，为所有电池供电的 ESP32 项目提供了可直接参考的实现范式。6165 Stars 的社区认可也证明了其在开源硬件领域的标杆地位。

---

📝 作者：蔡浩宇（jun-chy） / 📅 日期：2026-07-04 / 🔗 项目地址：[https://github.com/lmarzen/esp32-weather-epd](https://github.com/lmarzen/esp32-weather-epd)
