# ESP32-Plane-Radar 深度解析：用 1.28 寸圆形屏打造实时 ADS-B 航空雷达

> 📅 日期：2026-07-21　|　🏷️ 分类：ESP32　|　⭐ Stars：791　|　✍️ 作者：蔡浩宇（jun-chy）

## 项目链接

| 项目信息 | 内容 |
|---------|------|
| GitHub 地址 | https://github.com/MatixYo/ESP32-Plane-Radar |
| 作者 | MatixYo |
| 许可证 | MIT |
| 最近更新 | 2026-07-02 |
| 主要语言 | C++（Arduino 框架） |
| 创建时间 | 2026-05-31 |
| Fork 数 | 157 |
| 默认分支 | main |

## 一、项目简介与核心特性

ESP32-Plane-Radar 是一套运行在 **ESP32-C3 Super Mini** 与 **1.28 寸圆形 GC9A01 显示屏（240×240）** 上的开源固件。它以你设定的地理位置为中心，把周围天空中的实时飞机绘制成一张声呐风格的圆形雷达图。数据来源是免费的 ADS-B 开放数据平台 [adsb.fi](https://opendata.adsb.fi/)，设备通过 Wi-Fi 周期性拉取附近航班，再经坐标变换与双缓冲渲染呈现在圆形屏上。

ADS-B（Automatic Dependent Surveillance–Broadcast）是现代民航飞机自动广播自身位置、高度、速度、航向的协议。这个项目并没有自己搭接收链路去解调 1090MHz 信号，而是消费 ADS-B 聚合站的 REST API，把"看得见飞机"这件事压缩进一个口袋大小的桌面摆件。

| 特性 | 说明 |
|------|------|
| 实时航班 | 每 3 秒轮询 adsb.fi，最多渲染 64 架飞机 |
| 圆形雷达 UI | 同心环、十字准星、N/S/E/W 方位标 |
| 量程切换 | 5/10/15/25 km 四档预设，点按 BOOT 切换并写入 NVS |
| 飞机符号 | 红色航向三角 + 品红速度矢量，越界飞机显示边缘方位点 |
| 标签 | 呼号 / 机型 / 高度三行标签，自动避让屏幕中心 |
| 机场跑道叠加 | 内置 OurAirports 大型机场数据，青色跑道线 + ICAO 标签 |
| Wi-Fi 配网 | WiFiManager 强制门户，mDNS `plane-radar.local` |
| 双缓冲渲染 | 离屏 Sprite 合成后一次性推送，无闪烁 |
| 单按钮交互 | BOOT 键短按切量程、长按 3 秒清空配置重启 |

## 二、硬件架构

### 系统组成图

```
        ┌──────────────────────────────────────────────┐
        │              ESP32-C3 Super Mini              │
        │  ┌────────────┐    ┌───────────────────────┐  │
        │  │  RISC-V     │    │  WiFi 2.4GHz (STA/AP) │  │
        │  │  Core 160MHz│    │  └──► adsb.fi HTTPS   │  │
        │  └─────┬──────┘    └───────────┬───────────┘  │
        │        │                       │              │
        │   ┌────┴─────┐         ┌───────┴────────┐     │
        │   │ SPI Master│         │ NVS (Preferences)│     │
        │   │ 40MHz     │         │  量程/位置/单位    │     │
        │   └────┬─────┘         └────────────────┘     │
        │        │                                       │
        │   GPIO9 (BOOT) ── 按键 ── GND  (active LOW)    │
        └────────┼──────────────────────────────────────┘
                 │ SPI (MOSI/SCLK/CS/DC/RST)
        ┌────────▼─────────┐
        │  GC9A01 1.28"     │
        │  圆形 240×240     │
        │  IPS LCD (BGR)    │
        │  RGB565           │
        └───────────────────┘
```

整个系统以 ESP32-C3 为唯一主控：它既负责通过 Wi-Fi 向 adsb.fi 发起 HTTPS 请求、解析 JSON，又负责把结果做平面坐标变换并在 SPI 显示屏上绘制。所有用户配置（位置、量程、单位）持久化在 ESP32 的 NVS 分区，无需外部存储。

### 引脚配置（GC9A01 ↔ ESP32-C3 Super Mini）

引脚定义集中在 `include/config.h`，用 `constexpr gpio_num_t` 强类型表达，避免魔法数字：

| 功能 | 显示屏引脚 | ESP32-C3 GPIO | 说明 |
|------|-----------|---------------|------|
| 复位 | RST | GPIO 0 | 低电平复位 |
| 片选 | CS | GPIO 1 | SPI 从机选择 |
| 数据/命令 | DC | GPIO 10 | 0=命令 1=数据 |
| MOSI | SDA | GPIO 3 | SPI 主出从入 |
| 时钟 | SCL | GPIO 4 | SPI 时钟 |
| 用户按键 | BOOT | GPIO 9 | 低电平有效，短按切量程/长按重置 |

### 硬件清单

| 元件 | 规格 | 备注 |
|------|------|------|
| 主控板 | ESP32-C3 Super Mini | 4MB Flash，USB-C，原生 USB CDC |
| 显示屏 | 1.28 寸 GC9A01 圆形 IPS | 240×240，SPI，需反转+BGR |
| 电源 | USB-C 5V | 经板载 LDO 降至 3V3 |
| 外壳 | 3D 打印 STL | MakerWorld 提供模型 |

## 三、固件架构

### 文件结构表

| 路径 | 大小 | 模块职责 |
|------|------|---------|
| `src/main.cpp` | 2.7 KB | 入口：`setup()`/`loop()`，状态机调度 Wi-Fi 与雷达刷新 |
| `include/config.h` | 2.6 KB | 全局编译期常量：引脚、时序、默认位置、颜色 |
| `src/services/adsb_client.cpp` | 6.9 KB | ADS-B HTTP 拉取 + JSON 解析 + 飞机数据建模 |
| `src/services/wifi_setup.cpp` | 13.0 KB | WiFiManager 配网门户、自定义字段、重连 |
| `src/services/radar_location.cpp` | 2.0 KB | 雷达中心经纬度的 NVS 存取与校验 |
| `src/ui/radar_display.cpp` | 21.2 KB | 雷达渲染引擎：坐标变换、双缓冲、飞机绘制 |
| `src/ui/radar_range.cpp` | 3.5 KB | 量程预设状态机与 NVS 持久化 |
| `src/ui/runway_overlay.cpp` | 8.5 KB | 机场跑道线叠加绘制 |
| `src/ui/status_screens.cpp` | 7.3 KB | 配网/连接状态全屏提示页 |
| `partitions/plane_radar.csv` | 0.3 KB | 4MB Flash 分区表（单 app，无 OTA 槽） |
| `platformio.ini` | 0.7 KB | PlatformIO 构建配置与依赖 |

### 模块职责分层

固件采用清晰的 namespace 分层：`config::`（常量）、`services::adsb`（数据源）、`services::location`（位置）、`services::wifi`（网络）、`ui::radar`（渲染）。`main.cpp` 只做调度，把"取数据"和"画数据"解耦——这种分层让 ADS-B 数据源可以替换、渲染层可以独立测试。

## 四、核心代码深度分析

### 4.1 主循环状态机（src/main.cpp）

`main.cpp` 是整个固件的调度核心。它没有用 RTOS，而是用经典 Arduino 超级循环 + 时间戳比较实现非阻塞调度：

```cpp
void setup() {
  Serial.begin(115200);
  delay(500);
  Serial.println();
  Serial.println("Plane Radar");

  bootButtonInit();
  displayInit();
  if (wifiShowsSetupScreenOnBoot()) {
    statusScreenPortal();
  }
  services::location::init();
  ui::radar::rangeInit();
  services::adsb::setPollFn(wifiLoop);

  if (wifiSetupConnect()) {
    showRadarIfConnected();
  }
}
```

**逐段解析**：
- `bootButtonInit()` / `displayInit()`：先把用户输入和显示就绪，这样即使没连上网也能立刻在屏幕上给出反馈（配网提示页）。
- `wifiShowsSetupScreenOnBoot()`：判断是否需要首次配网。若需要，`statusScreenPortal()` 直接全屏显示 AP 名与门户地址，避免用户面对黑屏。
- `services::location::init()` 与 `ui::radar::rangeInit()`：分别从 NVS 读取上次保存的经纬度和量程档位。`init` 模式保证设备重启后状态一致。
- `services::adsb::setPollFn(wifiLoop)`：这是关键的解耦设计——把 `wifiLoop` 作为回调注入 ADS-B 客户端。因为后续 HTTPS 请求是阻塞的，注入回调让网络 I/O 期间也能持续喂 Wi-Fi 协议栈、扫描按键，避免连接超时。
- `wifiSetupConnect()`：尝试用已存凭据连 STA，成功后才 `showRadarIfConnected()` 进入雷达界面。

主循环 `loop()` 是一个三态状态机：

```cpp
void loop() {
  handleBootButton();
  wifiLoop();

  if (WiFi.status() != WL_CONNECTED) {
    if (g_radar_visible) {
      Serial.println("WiFi lost — will reconnect");
      g_radar_visible = false;
    }
    if (g_wifi_down_since == 0) {
      g_wifi_down_since = millis();
    }
    const unsigned long down_ms = millis() - g_wifi_down_since;
    if (down_ms >= config::kWifiDownGraceMs &&
        millis() - g_last_reconnect_ms >= config::kWifiReconnectIntervalMs) {
      g_last_reconnect_ms = millis();
      if (wifiReconnect()) {
        g_wifi_down_since = 0;
        showRadarIfConnected();
      }
    }
  } else {
    g_wifi_down_since = 0;
    if (!g_radar_visible) {
      showRadarIfConnected();
    } else if (millis() - g_last_adsb_fetch_ms >= config::kAdsbFetchIntervalMs) {
      g_last_adsb_fetch_ms = millis();
      fetchAndDrawAircraft();
    }
  }
  delay(10);
}
```

**逐段解析**：
- 先无条件 `handleBootButton()` 和 `wifiLoop()`，保证用户按键和网络保活始终响应。
- `if (WiFi.status() != WL_CONNECTED)` 分支处理断网：用 `g_wifi_down_since` 记录掉线起点，只有持续掉线超过 `kWifiDownGraceMs`（4 秒）才触发重连——这避免因瞬时丢包就重启配网门户。
- 重连还受 `kWifiReconnectIntervalMs`（15 秒）最小间隔节流，防止狂暴重连拖死主控。
- `else` 分支处理已联网：若雷达不可见则补画；否则用 `millis() - g_last_adsb_fetch_ms >= kAdsbFetchIntervalMs` 判断是否到 3 秒轮询时刻，到了就 `fetchAndDrawAircraft()`。
- 末尾 `delay(10)` 让出 CPU 给 Wi-Fi 协议栈后台任务。

注意所有计时都用 `millis()` 差值比较而非 `delay()`，这是嵌入式非阻塞设计的标准范式——单核 RISC-V 上，阻塞会直接卡死 Wi-Fi。

### 4.2 ADS-B 客户端：HTTPS 拉取与 JSON 解析（src/services/adsb_client.cpp）

这是数据源的核心。它要解决三个难题：阻塞 I/O 期间保活网络、流式读取大响应、容错解析字段不全的 JSON。

飞机数据结构定义在头文件，用定长数组避免动态内存：

```cpp
struct Aircraft {
  float lat;
  float lon;
  float nose_deg;      // 机头航向（用于三角符号朝向）
  float track_deg;     // 航迹角（用于速度矢量方向）
  float gs_knots;      // 地速（节）
  char callsign[9];    // 航班号
  char type[5];        // 机型代码
  char alt[12];        // 高度标签字符串
};
constexpr size_t kMaxAircraft = 64;
```

**逐段解析**：`nose_deg` 与 `track_deg` 分开存储是刻意设计——飞机机头朝向（true_heading）与实际航迹（track）在有侧风时不一致，渲染时三角符号按机头朝向、速度矢量按航迹角，更符合真实情况。`callsign[9]` 刚好放下最长的 ICAO 呼号（含结尾 `\0`）。`kMaxAircraft = 64` 是渲染性能与内存的折中——240×240 屏同时显示 64 架已接近可读极限。

URL 拼装与请求发起：

```cpp
bool fetchUpdate(double center_lat, double center_lon, float fetch_radius_km) {
  const float dist_nm = kmToNauticalMiles(fetch_radius_km);

  String url = kApiBase;
  url += String(center_lat, 6);
  url += "/lon/";
  url += String(center_lon, 6);
  url += "/dist/";
  url += String(dist_nm, 1);

  WiFiClientSecure client;
  client.setInsecure();

  HTTPClient http;
  if (!http.begin(client, url)) {
    Serial.println("adsb: http.begin failed");
    return false;
  }

  http.setTimeout(kRequestTimeoutMs);
  const int code = performGetWithPoll(http);
  if (code != HTTP_CODE_OK) {
    Serial.printf("adsb: HTTP %d\n", code);
    http.end();
    return false;
  }
```

**逐段解析**：
- adsb.fi 的 v3 API 形如 `/api/v3/lat/{lat}/lon/{lon}/dist/{nm}`，距离单位是海里，所以先 `kmToNauticalMiles` 换算（1 海里 = 1.852 km）。
- `client.setInsecure()` 跳过证书校验——ESP32-C3 上做完整 TLS 根证书校验要额外 Flash 且证书会过期，对于这种只读公开数据的场景，跳过是务实选择。
- `performGetWithPoll(http)` 是自封装的 GET，关键在于它会在重试间隙调用注入的 `s_poll_fn`（即 `wifiLoop`），让阻塞请求不致于饿死 Wi-Fi 协议栈。

带轮询的响应体读取是这个模块最精巧的部分：

```cpp
bool readResponseBodyWithPoll(HTTPClient& http, String& payload) {
  WiFiClient* stream = http.getStreamPtr();
  if (stream == nullptr) {
    return false;
  }

  const int content_length = http.getSize();
  if (content_length > 0) {
    payload.reserve(static_cast<unsigned>(content_length + 1));
  }

  uint8_t buffer[512];
  const unsigned long deadline = millis() + kRequestTimeoutMs;
  while (millis() < deadline) {
    pollNetwork();
    const int available = stream->available();
    if (available > 0) {
      const int to_read =
          available > static_cast<int>(sizeof(buffer)) ? static_cast<int>(sizeof(buffer))
                                                       : available;
      const int read_bytes = stream->readBytes(buffer, to_read);
      if (read_bytes > 0) {
        payload.concat(reinterpret_cast<const char*>(buffer),
                       static_cast<unsigned>(read_bytes));
      }
    }
    if (content_length > 0 &&
        static_cast<int>(payload.length()) >= content_length) {
      break;
    }
    if (!http.connected() && stream->available() <= 0) {
      break;
    }
    delay(1);
  }
  return payload.length() > 0;
}
```

**逐段解析**：
- `payload.reserve(content_length + 1)`：预分配容量，避免 `String` 在 `concat` 时反复 realloc 导致堆碎片——ESP32-C3 内存紧张，碎片化会让长时间运行后 malloc 失败。
- 用 512 字节固定缓冲 `buffer` 分块读取，而非一次性 `getString()`。这样每读一块就 `pollNetwork()` 一次，让 Wi-Fi 协议栈在数秒的下载过程中持续被喂。
- 两个退出条件：读够 `content_length` 字节，或连接已断且无残留数据。配合 `deadline` 超时兜底，三个条件保证不会无限阻塞。
- `delay(1)` 而非 `delay(10)`：下载期间需要高频轮询，但又不能 100% 占满 CPU，1ms 让步足够。

字段容错解析：不同 ADS-B 源对同一信息有多个可选字段名，项目用优先级回退处理：

```cpp
float pickNoseHeading(const JsonObject& plane) {
  float v = 0.0f;
  if (readJsonFloat(plane, "true_heading", &v)) {
    return v;
  }
  if (readJsonFloat(plane, "mag_heading", &v)) {
    return v;
  }
  if (readJsonFloat(plane, "track", &v)) {
    return v;
  }
  if (readJsonFloat(plane, "dir", &v)) {
    return v;
  }
  return 0.0f;
}
```

**逐段解析**：`pickNoseHeading` 按真实航向 → 磁航向 → 航迹 → 方向的优先级回退。真实航向最准（飞机仪表直接输出），磁航向需磁差修正但通常也可用，`track` 是 GPS 推算的航迹（受风影响），最后兜底 `dir`。这种"尽力而为"策略让固件能兼容多个 ADS-B 聚合站（adsb.fi、dump1090 等），换数据源无需改代码。

地面飞机过滤与高度标签格式化：

```cpp
bool isOnGround(const JsonObject& plane) {
  if (!plane["alt_baro"].is<const char*>()) {
    return false;
  }
  return strcmp(plane["alt_baro"].as<const char*>(), "ground") == 0;
}

void formatAltitudeTag(const JsonObject& plane, char* out, size_t out_len) {
  out[0] = '\0';
  if (out_len == 0) {
    return;
  }
  if (plane["alt_baro"].is<const char*>()) {
    const char* s = plane["alt_baro"].as<const char*>();
    if (strcmp(s, "ground") == 0) {
      strncpy(out, "GND", out_len - 1);
      out[out_len - 1] = '\0';
      return;
    }
  }
  float alt = 0.0f;
  if (readJsonFloat(plane, "alt_baro", &alt) ||
      readJsonFloat(plane, "alt_geom", &alt)) {
    snprintf(out, out_len, "%d ft", static_cast<int>(lroundf(alt)));
  }
}
```

**逐段解析**：ADS-B 里 `alt_baro` 字段是联合类型——飞机在地面时是字符串 `"ground"`，在空中时是数字（英尺）。`isOnGround` 先用 `is<const char*>()` 判类型再 strcmp，避免把数字当字符串解引用。`formatAltitudeTag` 同样先判类型：字符串则显示 `GND`，数字则 `lroundf` 四舍五入成 `%d ft`。这种基于 JSON 类型的分支是处理真实世界脏数据的典范。

主解析循环汇总：

```cpp
  size_t n = 0;
  for (JsonObject plane : ac) {
    if (n >= kMaxAircraft) {
      break;
    }
    if (!plane["lat"].is<float>() || !plane["lon"].is<float>()) {
      continue;
    }
    if (isOnGround(plane) && !config::kAdsbShowGroundAircraft) {
      continue;
    }

    s_aircraft[n].lat = plane["lat"].as<float>();
    s_aircraft[n].lon = plane["lon"].as<float>();
    s_aircraft[n].nose_deg = pickNoseHeading(plane);
    s_aircraft[n].track_deg = pickTrackHeading(plane);
    s_aircraft[n].gs_knots = pickGroundSpeed(plane);
    fillTagFields(&s_aircraft[n], plane);
    ++n;
  }
  s_aircraft_count = n;
```

**逐段解析**：遍历前先 `n >= kMaxAircraft` 截断，防止数组越界——这是嵌入式 C++ 必备的防御。`lat`/`lon` 缺失的记录直接 `continue`，没有位置的飞机无法绘制。地面飞机按配置 `kAdsbShowGroundAircraft`（默认 false）过滤，避免机场滑行的飞机糊满屏幕。最终 `s_aircraft_count` 记录有效数量，渲染层只画这么多。

### 4.3 雷达渲染引擎：坐标变换与双缓冲（src/ui/radar_display.cpp）

这是全项目最核心也最大的文件（21KB）。它把经纬度变成屏幕像素，再把飞机画上去。

**平面地球坐标变换**：

```cpp
constexpr float kKmPerDeg = 111.0f;

void offsetKmFromCenter(float lat, float lon, float* dx_km, float* dy_km,
                        float* dist_km) {
  *dx_km =
      static_cast<float>(lon - services::location::lon()) * kKmPerDeg;
  *dy_km =
      static_cast<float>(lat - services::location::lat()) * kKmPerDeg;
  *dist_km = sqrtf((*dx_km) * (*dx_km) + (*dy_km) * (*dy_km));
}
```

**逐段解析**：在小范围（几十公里）内，地球曲率可忽略，1° 经纬度 ≈ 111 km。`dx_km` 是东西偏移（经度差 × 111），`dy_km` 是南北偏移（纬度差 × 111）。距离用欧几里得范数。这种平面近似在 25km 量程内误差小于 0.1%，但对跨洲飞行会失真——所以项目把 fetch 半径限制在屏幕边缘对应的距离。注意这里没有做经度的纬度修正（高纬度经度圈更窄），因为在默认的温带纬度（阿姆斯特丹 52°N）和小量程下误差可接受。

经纬度转屏幕坐标：

```cpp
void latLonToScreen(float lat, float lon, int* out_x, int* out_y) {
  const float outer_km = radar::rangeCurrent().outer_km;
  const float px_per_km = static_cast<float>(radar::kGridOuterRadius) / outer_km;

  float dx_km = 0.0f;
  float dy_km = 0.0f;
  float dist_km = 0.0f;
  offsetKmFromCenter(lat, lon, &dx_km, &dy_km, &dist_km);

  *out_x = radar::kCenterX + static_cast<int>(lroundf(dx_km * px_per_km));
  *out_y = radar::kCenterY - static_cast<int>(lroundf(dy_km * px_per_km));
}
```

**逐段解析**：`px_per_km` 把"公里"换算成"像素"——外环半径 107 像素对应 `outer_km` 公里，所以缩放因子是 `107 / outer_km`。10km 档时 outer ≈ 13.3km，每公里约 8 像素。`out_y` 用减法是因为屏幕坐标 y 轴向下，而纬度向北递增，所以要反转。`lroundf` 四舍五入到整数像素，避免浮点绘制锯齿。

**越界飞机的边缘方位点**——这是 UX 上的巧妙设计：

```cpp
bool beyondRingEdgeDotFromLatLon(float lat, float lon, int* out_x, int* out_y) {
  float dx_km = 0.0f;
  float dy_km = 0.0f;
  float dist_km = 0.0f;
  offsetKmFromCenter(lat, lon, &dx_km, &dy_km, &dist_km);
  if (dist_km < 0.01f) {
    return false;
  }
  if (isInsideOuterRingKm(dist_km)) {
    return false;
  }

  const int cx = radar::kCenterX;
  const int cy = radar::kCenterY;
  const int rim_r = radar::kCenterX - radar::kBeyondRingScreenMarginPx;
  const float angle_rad = atan2f(dx_km, dy_km);

  *out_x = cx + static_cast<int>(lroundf(sinf(angle_rad) * rim_r));
  *out_y = cy - static_cast<int>(lroundf(cosf(angle_rad) * rim_r));
  return true;
}
```

**逐段解析**：当飞机在量程外（`isInsideOuterRingKm` 返回 false），不直接丢弃，而是计算它的真实方位角 `atan2f(dx_km, dy_km)`，然后把点钉在屏幕边缘 `rim_r` 半径处。这样用户能看到"东北方向有飞机飞来"，只是不知道具体多远。`atan2f(dx, dy)` 的参数顺序（x 在前）是因为航向角以正北为 0、顺时针递增，而 `atan2(y, x)` 默认以正东为 0——交换参数让 0° 朝北。`sinf`/`cosf` 再把角度还原成像素偏移，注意 y 仍是减法（屏幕坐标反转）。

**航向三角符号绘制**：

```cpp
void drawHeadingTriangle(int cx, int cy, float heading_deg, uint16_t color) {
  constexpr float kDegToRad = 0.01745329252f;
  const float rad = heading_deg * kDegToRad;
  const float sin_h = sinf(rad);
  const float cos_h = cosf(rad);

  int tip_x = 0;
  int tip_y = 0;
  noseTip(cx, cy, heading_deg, &tip_x, &tip_y);

  const int base_x =
      cx - static_cast<int>(lroundf(sin_h * static_cast<float>(radar::kAircraftTailLenPx)));
  const int base_y =
      cy + static_cast<int>(lroundf(cos_h * static_cast<float>(radar::kAircraftTailLenPx)));

  const int wing_x = static_cast<int>(lroundf(cos_h * radar::kAircraftTailHalfPx));
  const int wing_y = static_cast<int>(lroundf(sin_h * radar::kAircraftTailHalfPx));

  s_draw->fillTriangle(tip_x, tip_y, base_x + wing_x, base_y + wing_y,
                       base_x - wing_x, base_y - wing_y, color);
}
```

**逐段解析**：飞机符号是一个朝向航向的等腰三角形。`noseTip` 算出机头尖端（沿航向往前 `kAircraftNoseLenPx=8` 像素）。`base_x/base_y` 是机尾中点（沿航向往后 `kAircraftTailLenPx=3` 像素）。`wing_x/wing_y` 是垂直于航向的翼展半宽（`kAircraftTailHalfPx=4` 像素），用 `cos_h`/`sin_h` 旋转 90° 得到垂直方向。三个顶点（尖、左翼、右翼）送入 `fillTriangle` 实心填充。这里 `sin_h` 控制 x 分量、`cos_h` 控制 y 分量，正是"0°朝北、顺时针"坐标系下的标准旋转向量。整个绘制只用一次三角函数对，开销极低。

**速度矢量裁剪**——飞机速度线超出外环时要被裁掉：

```cpp
void clipPointToOuterRing(int x0, int y0, int* x1, int* y1) {
  const int max_r = radar::kGridOuterRadius;
  const int max_r_sq = max_r * max_r;
  if (distSqFromCenter(*x1, *y1) <= max_r_sq) {
    return;
  }

  const int dx = *x1 - x0;
  const int dy = *y1 - y0;
  float t = 1.0f;
  for (int step = 0; step < 20; ++step) {
    const int px = x0 + static_cast<int>(lroundf(dx * t));
    const int py = y0 + static_cast<int>(lroundf(dy * t));
    if (distSqFromCenter(px, py) <= max_r_sq) {
      *x1 = px;
      *y1 = py;
      return;
    }
    t -= 0.05f;
    if (t <= 0.0f) {
      *x1 = x0;
      *y1 = y0;
      return;
    }
  }
}
```

**逐段解析**：这不是解析几何的精确线段-圆相交，而是用步长 0.05 的线性插值二分逼近——从端点 `t=1.0` 往回退，每次退 5%，直到点落回圆内。20 步最多退到 `t=0`，精度足够（5% 像素级）。用 `distSqFromCenter`（平方距离）而非开方距离，省掉 `sqrtf`，是嵌入式优化的常规手段。这种近似法虽不精确，但对一条几像素的速度线视觉上无可挑剔。

**双缓冲无闪烁渲染**：

```cpp
// Double-buffered frame: composite the grid AND aircraft into the off-screen
// sprite, then blit it to the panel in a single pushSprite. Because the panel
// is updated in one pass, labels never show an erase/redraw gap — no flicker.
void renderFrame() {
  drawStaticGrid(s_frame);  // opens its own DrawScope(s_frame)
  {
    const DrawScope scope(s_frame);
    drawAircraft();
  }
  s_frame.pushSprite(0, 0);
  tft.setTextDatum(textdatum_t::top_left);
}
```

**逐段解析**：`s_frame` 是一个 240×240 的离屏 Sprite（占用约 112KB RAM）。`drawStaticGrid` 先把网格、跑道、方位标画进 Sprite；`DrawScope scope(s_frame)` 是 RAII 守卫，把全局绘图目标 `s_draw` 临时切到 Sprite，作用域结束自动还原——这种设计让所有绘图函数无需关心目标，同一份代码既能画到屏幕也能画到缓冲。最后 `pushSprite(0,0)` 一次性把整帧 SPI 推送到面板。因为面板在一帧内完整刷新，不会出现"先擦后画"的中间态，彻底消除闪烁。若 RAM 不够分配 Sprite（`ensureFrameSprite` 返回 false），回退到直绘屏幕，保证可用性。

### 4.4 量程状态机与 NVS 持久化（src/ui/radar_range.cpp）

量程档位用编译期数组表达，运行时只存索引：

```cpp
constexpr RangePreset kRangePresets[] = {
    {5.0f, 5.0f * kRing3ToOuterKm},
    {10.0f, 10.0f * kRing3ToOuterKm},
    {15.0f, 15.0f * kRing3ToOuterKm},
    {25.0f, 25.0f * kRing3ToOuterKm},
};
```

`ring3_km` 是第 3 环（最外可视环）标注的距离，`outer_km` 是飞机数学坐标的外环距离 = `ring3_km × 4/3`（因为第 3 环是外半径的 3/4 处）。切换时：

```cpp
void rangeNext() {
  s_range_index = static_cast<uint8_t>((s_range_index + 1) % kRangePresetCount);
  saveRangeIndex();
}
```

取模循环 0→1→2→3→0，并立即写 NVS。fetch 半径随量程自适应：

```cpp
float fetchRadiusKm() {
  const float outer_km = rangeCurrent().outer_km;
  const float screen_r_px =
      static_cast<float>(kCenterX - kBeyondRingScreenMarginPx);
  return outer_km * (screen_r_px / static_cast<float>(kGridOuterRadius));
}
```

**逐段解析**：fetch 半径 = `outer_km × (屏幕半径 / 网格外环半径)`。因为屏幕边缘比网格外环（107px）略大（120-2=118px），fetch 范围会比可视范围稍大，保证边缘方位点有数据可画。这个比例换算让"看得见的区域"和"拉取的区域"精确匹配，既不浪费带宽也不漏掉边缘飞机。

## 五、API 使用指南

项目消费的是 adsb.fi 的公开 REST API，你可以独立验证。给定经纬度和半径（海里），返回附近飞机的 JSON：

```bash
# 查询阿姆斯特丹（52.3676, 4.9041）周围 5 海里内的飞机
curl -s "https://opendata.adsb.fi/api/v3/lat/52.3676/lon/4.9041/dist/5" | python3 -m json.tool | head -40
```

返回结构（节选）：

```json
{
  "ac": [
    {
      "hex": "484506",
      "flight": "KLM123  ",
      "lat": 52.367,
      "lon": 4.905,
      "alt_baro": 35000,
      "gs": 450,
      "true_heading": 90.2,
      "track": 92.1,
      "t": "B738"
    }
  ]
}
```

Python 示例（解析并计算飞机相对方位）：

```python
import requests, math

LAT, LON = 52.3676, 4.9041
url = f"https://opendata.adsb.fi/api/v3/lat/{LAT}/lon/{LON}/dist/10"
data = requests.get(url, timeout=10).json()

for ac in data.get("ac", []):
    if "lat" not in ac or "lon" not in ac:
        continue
    dx = (ac["lon"] - LON) * 111.0  # km, 东西
    dy = (ac["lat"] - LAT) * 111.0  # km, 南北
    dist = math.hypot(dx, dy)
    bearing = math.degrees(math.atan2(dx, dy)) % 360  # 0=北, 顺时针
    callsign = ac.get("flight", ac.get("hex", "?")).strip()
    print(f"{callsign:8} 距离 {dist:5.1f}km  方位 {bearing:5.1f}°  "
          f"高度 {ac.get('alt_baro')}ft  地速 {ac.get('gs')}kt")
```

这段 Python 与固件 `adsb_client.cpp` 的逻辑完全对应：平面近似算 dx/dy、`atan2(dx, dy)` 算方位、字段回退取 callsign。你可以用它先验证某地是否有航班数据，再决定是否值得把坐标烧进固件。

## 六、编译与部署

### 环境搭建

| 项 | 要求 |
|----|------|
| PlatformIO Core | ≥ 6.x（或 VSCode 插件） |
| 平台 | espressif32@6.5.0 |
| 框架 | Arduino |
| 依赖 | LovyanGFX、WiFiManager、ArduinoJson（`platformio.ini` 自动拉取） |

### 编译配置（platformio.ini 关键项）

```ini
[env:supermini]
platform = espressif32@6.5.0
board = esp32-c3-devkitm-1
framework = arduino
monitor_speed = 115200
board_build.partitions = partitions/plane_radar.csv
extra_scripts = post:scripts/merge_firmware.py
board_build.embed_files = data/ui_font.vlw
build_flags =
  -std=gnu++17
  -DARDUINO_USB_MODE=1
  -DARDUINO_USB_CDC_ON_BOOT=1
  -DWM_NODEBUG
  -DWM_MDNS
```

`-DARDUINO_USB_CDC_ON_BOOT=1` 让 Super Mini 的原生 USB 作为串口，无需外接 UART；`-DWM_MDNS` 启用 WiFiManager 的 mDNS，才能用 `plane-radar.local` 访问；`board_build.embed_files` 把 VLW 矢量字体嵌入固件镜像。

### Flash 分区表（partitions/plane_radar.csv）

```
# 4 MB flash — single large app (no OTA slot). Plane Radar + runway dataset.
nvs,      data, nvs,     0x9000,  0x5000,
otadata,  data, ota,     0xe000,  0x2000,
app0,     app,  ota_0,   0x10000, 0x300000,
spiffs,   data, spiffs,  0x310000,0xE0000,
coredump, data, coredump,0x3F0000,0x10000,
```

app 分区给到 3MB（0x300000），是因为内嵌了 142KB 的全球大型机场数据 `large_airports_data.cpp`。不做双 OTA 槽以最大化 app 空间。

### 烧录步骤

```bash
# 1. 克隆并编译上传（USB 连接 Super Mini）
git clone https://github.com/MatixYo/ESP32-Plane-Radar.git
cd ESP32-Plane-Radar
pio run -t upload
pio device monitor    # 115200，观察启动日志

# 2. 生成可网页烧录的合并镜像（无需 PlatformIO 也能刷）
chmod +x scripts/merge-firmware.sh
./scripts/merge-firmware.sh
# 产出 release/plane-radar-merged.bin，用 esptool-js 在 Chrome/Edge 烧到 0x0
```

首次上电：连接 `PlaneRadar-Setup` Wi-Fi → 浏览器开 `http://plane-radar.local` → 填家庭 Wi-Fi、经纬度 → 保存自动重启进入雷达。

## 七、项目亮点与适用场景

**亮点**：
- **双缓冲无闪烁**：112KB 离屏 Sprite 一次推送，圆形屏动画顺滑。
- **阻塞 I/O 保活**：把 `wifiLoop` 注入 HTTP 读取循环，单核也能边下载边喂协议栈。
- **边缘方位点设计**：量程外的飞机不丢弃，钉在屏幕边缘示向，信息密度最大化。
- **字段回退容错**：多数据源、字段缺失、联合类型全覆盖，换 ADS-B 源零改码。
- **单按钮全交互**：BOOT 键短按切量程、长按重置，无额外按键成本。

**适用场景**：航空爱好者桌面摆件、窗台航班 spotting 辅助、STEM 航空教育教具、ESP32 图形渲染与 HTTPS JSON 处理的学习样板。

## 八、总结

ESP32-Plane-Radar 把"看见头顶的飞机"这件事做到了口袋大小。它的工程价值不在 ADS-B 协议本身（它消费 API 而非解调信号），而在于如何在 4MB Flash、有限 RAM 的 RISC-V 单片机上，优雅地完成 HTTPS 流式下载、JSON 容错解析、平面坐标变换、双缓冲图形渲染这一整条链路。尤其值得借鉴的是它对"阻塞"的处理——用注入回调把网络保活织进 I/O 等待，以及用平面地球近似把球面坐标降维成像素坐标的工程取舍。对于想学 ESP32 图形与联网综合项目的开发者，这是一个体量适中、注释充分、可读性极高的范本。

---

📝 作者：蔡浩宇（jun-chy）　|　📅 日期：2026-07-21　|　🔗 项目地址：https://github.com/MatixYo/ESP32-Plane-Radar
