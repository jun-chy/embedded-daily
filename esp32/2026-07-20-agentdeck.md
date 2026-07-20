# AgentDeck：ESP32 AI 编码代理物理控制面板固件深度解析

> **日期**：2026-07-20 ｜ **分类**：ESP32 / FreeRTOS / WebSocket / AI Agent ｜ **Stars**：154 ｜ **作者**：蔡浩宇（jun-chy）

## 项目链接

| 项目信息 | 内容 |
|---------|------|
| GitHub 地址 | [puritysb/AgentDeck](https://github.com/puritysb/AgentDeck) |
| 作者 | puritysb |
| 许可证 | MIT |
| 最近更新 | 2026-07-20 |
| 主要语言 | C |
| 代码规模 | ~605 MB（含 Apple/Android/Esp32 多平台 monorepo） |
| 默认分支 | master |
| Topics | ai-agents、esp32、claude-code、stream-deck、iot、swiftui、monorepo |

## 项目简介

AgentDeck 是一套面向 AI 编码代理（Claude Code、Codex CLI、OpenCode、OpenClaw）的**物理控制面板**——把"和 AI 聊天"变成"用混音台驾驶 AI"。它最初以 Elgato Stream Deck+ 为载体，现已扩展到 **22 个显示面**并行运行：平板、电子墨水屏、手机、ESP32 多形态屏幕、LED 矩阵、终端 TUI 等。

本文聚焦其中的 **ESP32 固件端**（`esp32/` 目录），它把 ESP32 开发板变成一块 WiFi 恒亮的"代理仪表盘"：通过 mDNS 自动发现桥接守护进程、经 WebSocket 实时同步代理状态、用 LVGL 在彩屏上渲染水族箱式动画，并在 LED 矩阵/e-ink 上做轻量呈现。固件基于 Arduino + FreeRTOS 双核架构，支持 8 种开发板，含 OTA 升级、串行桥接、WiFi 配网门户、唤醒词等完整能力。

### 核心特性一览

| 特性 | 说明 |
|------|------|
| FreeRTOS 双核架构 | Core 0 跑网络任务（WiFi + mDNS + WebSocket + 串行 JSON），Core 1 跑 UI 渲染（LVGL 或 LED 矩阵），互斥量保护共享状态 |
| 串行 + WiFi 双传输 | USB 串行 JSON 为主、WiFi WebSocket 为辅；串行激活时自动"停泊"无线射频省 2.4GHz 带宽 |
| mDNS 自动发现 | 浏览 `_agentdeck._tcp`，优先选 `agent=daemon` TXT 记录的守护端点，解析 token/project 元数据 |
| WebSocket 状态机 | 指数退避重连（3s→8s 封顶）、心跳 ping/pong、UI 核→网络核出队队列（线程安全） |
| JSON 协议解析 | ArduinoJson 解析 `state_update`/`usage_update`/`sessions_list`/`timeline` 等消息，UTF-8 截断安全 |
| 生物态状态机 | `AgentState` → `CreatureState` 派生（章鱼 SLEEPING/FLOATING/WORKING/ASKING），按代理状态驱动动画 |
| 多板 HAL | 8 块板（Round AMOLED、IPS 3.5"、B86 Box、TTGO、C6 1.47"、IPS 10.1"、TC001、InkDeck）统一板级头文件 + 编译宏 |
| WiFi 配网 | WiFiManager 非阻塞 captive portal + NVS 凭据持久化 + NTP 时间同步（用于配额重置倒计时） |
| OTA 升级 | 分区表 + base64 分块接收 + Update 库写入，`agentdeck esp32-ota` 远程推送 |
| 唤醒词 | ESP32 端 microWakeWord（TFLite），`openclaw_wake_word.tflite` 本地推理 |

## 硬件架构

AgentDeck 的 ESP32 端运行在多种开发板上，核心是"ESP32 SoC + 显示屏 + WiFi"三件套，外加以 USB 串行或 WiFi 作为与桥接守护进程的通道。

### 系统组成图

```
        ┌─────────────────────────────────────────────┐
        │        宿主机（macOS / Windows / Linux）       │
        │  ┌─────────────┐    ┌──────────────────┐    │
        │  │ claude CLI  │    │ AgentDeck Daemon │    │
        │  │ codex/opencode│◄──│  (port 9120)     │    │
        │  └─────────────┘ PTY└────────┬─────────┘    │
        │    Claude Code Hooks (HTTP)  │  mDNS 广播     │
        └──────────────────────────────┼───────────────┘
                                       │
                ┌──────────────────────┼──────────────────┐
                │ USB 串行 JSON        │ WiFi WebSocket     │
                │ (主传输, 5V 供电)     │ (wss/?token=...)   │
                │                      │                    │
        ┌───────▼──────────────────────▼──────────────┐    │
        │              ESP32 显示板固件                │    │
        │  ┌────────────── Core 0 ──────────────┐     │    │
        │  │ Serial ←→ WiFiManager ←→ mDNS       │     │    │
        │  │          ↓                          │     │    │
        │  │       WebSocket Client              │     │    │
        │  │          ↓ (JSON)                    │     │    │
        │  │     Protocol 解析 → g_state         │     │    │
        │  └──────────────┬───────────────────────┘     │    │
        │                 │ 互斥量 g_stateMutex           │    │
        │  ┌──────────────▼ Core 1 ────────────────┐     │    │
        │  │  LVGL 渲染 / LED 矩阵分页 / e-ink     │     │    │
        │  │  水族箱动画(章鱼/小龙虾/霓虹灯鱼)       │     │    │
        │  │  HUD(5h/7d 配额、工具、时间线)         │     │    │
        │  └───────────────────────────────────────┘     │    │
        └───────────────────┬───────────────────────────┘    │
                            │ SPI / I2C / RMT / MIPI-DSI      │
        ┌───────────────────▼───────────────────────────┐    │
        │  显示屏：AMOLED / IPS / LED 矩阵 / e-ink      │    │
        └───────────────────────────────────────────────┘    │
```

### 硬件清单表

| 开发板 | SoC | 屏幕 | 分辨率 | 传输 | 特殊能力 |
|--------|-----|------|--------|------|----------|
| Round AMOLED 1.8" | ESP32-S3 | 圆形 AMOLED（ST77916） | 360×360 | WiFi OTA | 圆形 UI |
| IPS LCD 3.5" | ESP32-S3 | 矩形 IPS | 480×320 | WiFi OTA | 触摸 |
| B86 Box 4" | ESP32-S3 | 壁挂触摸面板 | 480×480 | WiFi OTA | 一次性 USB 分区迁移 |
| TTGO T-Display 1.14" | ESP32（经典） | LilyGO ST7789 TFT | 135×240 | WiFi OTA | 物理按钮旋转 |
| Waveshare LCD 1.47" | ESP32-C6 | ST7789 TFT | 172×320 | WiFi OTA | 单核 C6 |
| IPS 10.1" | ESP32-P4 + C6 | Guition JD9365 MIPI-DSI | 800×1280 | WiFi OTA | ESP-Hosted SDIO 双芯 |
| Ulanzi TC001 | ESP32（经典） | 8×32 WS2812B LED | 256 像素 | WiFi OTA | RMT 驱动 LED |
| InkDeck | XIAO ESP32-S3 Plus | Seeed 7.5" e-ink（UC8179） | 800×480 | WiFi OTA | 局部刷新 |

### 引脚与板级配置

每块板用一个头文件（`esp32/boards/board_*.h`）描述引脚、屏幕驱动、SPI/I2C 总线、背光、触摸等，由 `board_config.h` 汇总并通过 PlatformIO 编译宏（`BOARD_IPS35`、`BOARD_AMOLED`、`BOARD_LED8X32`、`BOARD_IPS10`、`BOARD_TTGO`、`BOARD_ESP32_C6_147`、`BOARD_INKDECK`）选择。例如 TTGO 用 `BOARD_PIN_BTN1 = GPIO35`（仅输入、外部上拉）做旋转按钮，C6 用 `BOOT = GPIO9`（内部上拉）。

## 固件架构

### 文件结构表

| 路径 | 模块 | 职责 |
|------|------|------|
| `src/main.cpp` | 主入口 | FreeRTOS 双核任务创建、网络主循环、UI 主循环、按钮/手势处理 |
| `src/config.h` | 配置常量 | 端口、退避、LVGL tick、生物数量上限、FreeRTOS 栈大小、板级宏 |
| `src/state/agent_state.h` | 状态机 | `DashboardState` 全局结构、`AgentState`/`CreatureState` 枚举、派生逻辑、互斥量 |
| `src/net/ws_client.cpp/.h` | WebSocket 客户端 | 连接管理、事件回调、指数退避、出队队列、命令发送 |
| `src/net/protocol.cpp/.h` | 协议解析 | ArduinoJson 解析桥接消息、状态/用量/会话/时间线更新、OTA 接收 |
| `src/net/wifi_manager.cpp/.h` | WiFi 管理 | WiFiManager 配网门户、NVS 持久化、NTP 同步、射频停泊、IPS10 特例 |
| `src/net/mdns_discovery.cpp/.h` | mDNS 发现 | 浏览 `_agentdeck._tcp`、TXT 记录解析、守护端点优先 |
| `src/net/serial_client.cpp/.h` | 串行桥接 | USB JSON 行协议、设备信息上报 |
| `src/ui/display.cpp/.h` | 显示抽象 | LVGL 初始化、屏幕加载、方向切换 |
| `src/ui/screens/*.cpp` | 屏幕 | Splash / Aquarium / Settings / Permission / TTGO Overlay |
| `src/ui/terrarium/*.cpp` | 生物动画 | 章鱼、小龙虾、霓虹灯鱼、气泡、水草、云、粒子 |
| `src/ui/matrix/*.cpp` | LED 矩阵 | TC001 8×32 分页 UI、点阵字形 |
| `src/ui/eink/*.cpp` | 电子墨水 | InkDeck 1-bit 帧渲染、局部刷新 |
| `src/audio/wake_word.cpp` | 唤醒词 | microWakeWord TFLite 推理 |
| `src/util/ota_capability.cpp` | OTA | 分块接收、Update 库写入 |
| `boards/board_*.h` | 板级 HAL | 引脚、屏幕驱动、总线、背光 |
| `partitions/*.csv` | 分区表 | 双 OTA 分区、NVS、PHY |

### 模块职责

固件遵循"**双核隔离 + 单一全局状态 + 互斥量同步**"的设计：网络核负责所有 I/O（串行、WiFi、mDNS、WebSocket），把解析后的数据写入受 `g_stateMutex` 保护的 `g_state`；UI 核只读 `g_state` 渲染画面，用户交互产生的出站命令（批准/拒绝/选选项）入队 `outbox`，由网络核在 `pumpOutbound()` 中统一发送，避免 arduinoWebSockets 的线程安全问题。

## 核心代码深度分析

以下代码均取自项目真实源码（`master` 分支），逐段解析其设计意图与实现细节。

### 1. 双核任务划分与全局状态（main.cpp）

固件入口声明了贯穿两核的共享状态与互斥量：

```cpp
// ===== Global state =====
DashboardState g_state;
SemaphoreHandle_t g_stateMutex = nullptr;
```

**解析**：
- `g_state` 是唯一的全局状态结构（定义见 `agent_state.h`），承载连接、代理状态、用量、会话、时间线、生物态等所有可渲染数据。它被两个核同时访问，因此必须配互斥量。
- `g_stateMutex` 在 `setup()` 中由 `xSemaphoreCreateMutex()` 创建。所有读写都经 `lockState()`/`unlockState()` 这对内联函数（见 `agent_state.h` 末尾）完成，本质是 `xSemaphoreTake(..., portMAX_DELAY)` / `xSemaphoreGive`。

接下来是网络核任务的"骨架"，它定义了整个连接生命周期的优先级：

```cpp
static void networkTask(void* param) {
    Serial.printf("[Net] Task started on core %d\n", xPortGetCoreID());
    // 1. Serial JSON listener (always active — USB is always connected)
    Net::serialInit();
    // 2. Connect WiFi (non-blocking attempt)
    Net::wifiInit();
    // 3. Start mDNS discovery
    Net::mdnsInit();
    // 4. Init WebSocket
    Net::wsInit();
```

**逐行解析**：
- 任务先打印所在核（应为 Core 0，由 `config.h` 的 `CORE_NETWORK = 0` 钉死）。
- **顺序极关键**：串行监听 → WiFi → mDNS → WebSocket。串行之所以排第一且"always active"，是因为 USB 永远在线、是最可靠的传输；即便 WiFi 没配好，串行 JSON 仍能让板子工作。
- `wifiInit()` 设为非阻塞（下文 `wifi_manager.cpp` 详述），无凭据时起 captive portal 但立即返回，不卡住后续 mDNS/WS 初始化。

进入主循环后，是一段精心设计的"连接自愈"逻辑：

```cpp
    while (true) {
        // === Always poll serial (USB JSON from bridge) ===
        Net::serialLoop();
        // === WiFi portal (non-blocking, processes captive portal if active) ===
        Net::wifiLoop();
```

**解析**：循环每轮（末尾 `vTaskDelay(pdMS_TO_TICKS(10))` 即 10ms）先轮询串行、再处理 WiFi 门户事件。这两步始终执行，保证 USB 桥接永远可用。

最精妙的是 mDNS 发现与重连的协同：

```cpp
        // === Continuous mDNS discovery ===
        // Only perform mDNS polling if we are NOT connected.
        // Constant mDNS querying while connected consumes CPU, Wi-Fi bandwidth,
        // and induces severe packet jitter/latency spikes on ESP32, leading to disconnects.
        if (!Net::wifiRadioParked() && Net::wifiConnected() && !Net::wsConnected() && !Net::wsConnecting() && Net::mdnsPoll(bridge)) {
            bool ipChanged = (strcmp(currentBridgeIp, bridge.ip) != 0) || (currentBridgePort != bridge.port);
            if (ipChanged || !Net::wsConnected()) {
                if (ipChanged) {
                    Serial.printf("[Net] Bridge (re)discovered via mDNS: %s:%d\n", bridge.ip, bridge.port);
                    strncpy(currentBridgeIp, bridge.ip, sizeof(currentBridgeIp) - 1);
                    currentBridgePort = bridge.port;
                    // Self-heal the persisted endpoint ...
                    Net::wifiSaveProvisionedBridge(bridge.ip, bridge.port, bridge.token);
                    if (Net::wsConnected()) Net::wsDisconnect();
                }
                lockState();
                strncpy(g_state.bridgeIp, bridge.ip, sizeof(g_state.bridgeIp) - 1);
                g_state.bridgePort = bridge.port;
                strncpy(g_state.authToken, bridge.token, sizeof(g_state.authToken) - 1);
                unlockState();
```

**逐行解析**：
- 注释点出一条 ESP32 实战经验：**连着的时候持续发 mDNS 查询会引发严重抖动甚至掉线**。所以只在"未连接"时才 poll。
- 条件 `!wifiRadioParked && wifiConnected && !wsConnected && !wsConnecting` 四重门控，避免在射频停泊、WiFi 未就绪、或正在重连时重复触发。
- `mdnsPoll()` 返回 `true` 表示发现了新端点。接着比较 IP/端口是否变化——守护进程可能因 DHCP 漂移换了地址。
- 注释里 `67934f94` 指名道姓引用了一个历史 commit 的 bug：旧代码把桥接 IP 存了 NVS 却从不刷新，导致守护进程搬家后板子每次重启都连旧 IP 死循环。这里 `wifiSaveProvisionedBridge()` 把新发现的端点写回 NVS，实现跨重启自愈。这种"注释即 blame"的风格是真实工程代码的标志。
- IP 变了就先 `wsDisconnect()` 旧连接，让 `wsConnect` 干净地重新绑定。
- 最后在互斥量保护下把端点写入 `g_state`，供 UI 显示"已连接到 xxx:9120"。

### 2. WebSocket 客户端：事件回调与指数退避（ws_client.cpp）

WebSocket 是 WiFi 板的生命线。`onWsEvent` 是所有 WS 事件的入口：

```cpp
static void onWsEvent(WStype_t type, uint8_t* payload, size_t length) {
    switch (type) {
        case WStype_DISCONNECTED:
            Serial.println("[WS] Disconnected");
            connected = false;
            connecting = false;
            lockState();
            // Only mark disconnected if serial is also not connected
            // (serial data is authoritative — don't override it)
            if (!Net::serialConnected()) {
                g_state.markBridgeDisconnected();
            }
            unlockState();
            break;
```

**逐行解析**：
- `WStype_DISCONNECTED` 分支先把 `connected`/`connecting` 两个状态位清零。
- **关键设计**：调用 `markBridgeDisconnected()` 前先检查 `!serialConnected()`。因为串行是"权威传输"——如果 USB 还在供数据，WS 掉了不应让 UI 显示"已断开"（章鱼变 SLEEPING）。只有两条传输都断了才真正标记断开。这避免了插着 USB 时 WiFi 抖动导致的画面闪烁。
- `markBridgeDisconnected()`（见 `agent_state.h`）会清空会话/用量等易失数据，但保留 `lastMessageMs` 等连接记忆。

连接成功时的握手与自报家门：

```cpp
        case WStype_CONNECTED:
            Serial.printf("[WS] Connected to %s:%d\n", savedIp, savedPort);
            connected = true;
            connecting = false;
            reconnectMs = WS_RECONNECT_MIN_MS;
            ws.setReconnectInterval(reconnectMs);
            lockState();
            g_state.wsConnected = true;
            g_state.lastMessageMs = millis();
            unlockState();
            // Request initial state + identify ourselves (device_info is
            // request-driven on serial, but nothing requests it over WS — a
            // WiFi-only board must announce or the daemon never learns its
            // board/buildHash).
            Net::wsSendCommand("query_usage");
            Protocol::announceDeviceInfo();
            break;
```

**逐行解析**：
- 连上后立刻把退避值 `reconnectMs` 重置回 `WS_RECONNECT_MIN_MS`（3000ms），并同步到库的内部定时器。这样下次掉线从最小退避重新开始。
- 注释点出串行与 WS 的不对称：串行板由守护进程主动 `device_info_request`，但 WS 上没人来问，所以 WiFi-only 板子（如 InkDeck）必须**主动 `announceDeviceInfo()`** 自报板型/构建哈希，否则守护进程永远不知道连进来的是什么板。
- `wsSendCommand("query_usage")` 立即拉取一次用量数据，避免 UI 空白。

出队队列是跨核通信的关键，UI 核绝不能直接调 WS：

```cpp
// ── outbound queue (UI core → network core) ──
// LVGL event callbacks run on CORE_UI; the WebSocket + serial transports are
// driven from CORE_NETWORK. arduinoWebSockets is not thread-safe, so UI-side
// senders enqueue here and the network task drains via pumpOutbound().
static constexpr int OUTBOX_MAX = 6;
static constexpr int OUTBOX_LEN = 200;
static char outbox[OUTBOX_MAX][OUTBOX_LEN];
static int outboxHead = 0;
static int outboxCount = 0;
static SemaphoreHandle_t outboxMutex = nullptr;
```

**解析**：
- `arduinoWebSockets` 不是线程安全的，若 UI 核（Core 1）的 LVGL 按钮回调直接 `ws.sendTXT()`，会和网络核的 `ws.loop()` 竞争导致崩溃。这里用一个**固定大小的环形缓冲 + 互斥量**解耦：UI 核只入队，网络核只出队。
- `OUTBOX_MAX = 6`、`OUTBOX_LEN = 200` 是有意的小——交互命令是用户节奏的（点一下按钮），不是突发流；满了就丢，符合"宁可丢命令也不要堆积导致 UI 卡顿"的实时性取舍。

入队与出队的实现：

```cpp
void queueOutbound(const char* json) {
    if (!json || !json[0] || !outboxMutex) return;
    xSemaphoreTake(outboxMutex, portMAX_DELAY);
    if (outboxCount < OUTBOX_MAX) {
        int idx = (outboxHead + outboxCount) % OUTBOX_MAX;
        strncpy(outbox[idx], json, OUTBOX_LEN - 1);
        outbox[idx][OUTBOX_LEN - 1] = '\0';
        outboxCount++;
    }
    xSemaphoreGive(outboxMutex);
}
void pumpOutbound() {
    if (!outboxMutex) return;
    while (true) {
        char line[OUTBOX_LEN];
        xSemaphoreTake(outboxMutex, portMAX_DELAY);
        if (outboxCount == 0) { xSemaphoreGive(outboxMutex); break; }
        strncpy(line, outbox[outboxHead], sizeof(line));
        line[sizeof(line) - 1] = '\0';
        outboxHead = (outboxHead + 1) % OUTBOX_MAX;
        outboxCount--;
        xSemaphoreGive(outboxMutex);
        if (connected) ws.sendTXT(line);
        else Net::serialWriteJsonLine(line);  // serial bridge consumes line-delimited JSON
    }
}
```

**逐行解析**：
- `queueOutbound`：取互斥量 → 没满就写到 `(head + count) % MAX` 位置（环形队列尾）→ 释放。`strncpy` 后手动补 `\0` 防溢出。
- `pumpOutbound`：在 `while(true)` 里循环取一条，**关键是在 `xSemaphoreGive` 之后再发**（`ws.sendTXT` 可能阻塞在网络栈里，不能持有互斥量，否则 UI 核入队会被卡死）。
- 三元决策：连着就走 WS，没连就走串行行协议 `serialWriteJsonLine`。这让"插 USB 时按按钮"也能工作。

连接与退避的核心：

```cpp
void wsConnect(const char* ip, uint16_t port, const char* token) {
    if (connected || connecting) {
        return;
    }
    connecting = true;
    ws.disconnect();
    delay(10);
    strncpy(savedIp, ip, sizeof(savedIp) - 1);
    savedPort = port;
    strncpy(savedToken, token, sizeof(savedToken) - 1);
    char path[104];
    if (token[0] != '\0') {
        snprintf(path, sizeof(path), "/?token=%s&clientType=esp32", token);
    } else {
        strcpy(path, "/?clientType=esp32");
    }
    ws.begin(ip, port, path);
    ws.onEvent(onWsEvent);
    ws.setReconnectInterval(reconnectMs);
    ws.enableHeartbeat(WS_PING_INTERVAL_MS, WS_PONG_TIMEOUT_MS, 2);
    Serial.printf("[WS] Connecting to %s:%d\n", ip, port);
}
```

**逐行解析**：
- 幂等保护：`if (connected || connecting) return;` 防止 mDNS poll 重复触发连接。
- 先 `disconnect()` 再 `delay(10)`，给底层 socket 时间释放，避免端口占用。
- **token 通过 URL query 传递**（`/?token=...&clientType=esp32`）：这匹配 README 里"LAN 客户端需 auth token"的安全模型。`clientType=esp32` 让守护进程把它归类为 ESP32 设备（而非 Stream Deck/Android），决定推送的帧格式。
- `enableHeartbeat(15s ping, 30s pong timeout, 2)`：每 15s 发 ping，30s 收不到 pong 视为断连，最多 2 次未 pong 才断。这三个常量在 `config.h` 定义。

退避策略在 `wsLoop` 里动态推进：

```cpp
void wsLoop() {
    if (!WiFi.isConnected()) {
        if (connected || connecting) {
            ws.disconnect();
        }
        connected = false;
        connecting = false;
        return;
    }
    ws.loop();
    if (!connected && savedIp[0] != '\0') {
        uint32_t now = millis();
        if (now - lastReconnectAttempt > reconnectMs) {
            lastReconnectAttempt = now;
            uint32_t next = reconnectMs * 2;
            if (next > WS_RECONNECT_MAX_MS) next = WS_RECONNECT_MAX_MS;
            reconnectMs = next;
            ws.setReconnectInterval(reconnectMs);
        }
    }
}
```

**逐行解析**：
- 开头的 WiFi 检查是防御性的：WiFi 断了就强制断 WS 并清状态，避免在没网时 `ws.loop()` 空转。
- `millis() - lastReconnectAttempt > reconnectMs` 触发退避翻倍，`reconnectMs * 2` 上限 `WS_RECONNECT_MAX_MS`（8000ms）。3s → 6s → 8s（封顶）。每次都 `setReconnectInterval` 同步给库——注释强调"必须推进新值，否则库卡在 `wsConnect` 时的旧值"。
- 指数退避避免了狂连守护进程，在 2.4GHz 拥堵时尤其重要。

### 3. 协议解析：状态机与用量更新（protocol.cpp）

`protocol.cpp` 是 JSON 协议的核心。先看代理状态的字符串→枚举映射：

```cpp
static AgentState parseState(const char* s) {
    if (!s) return AgentState::DISCONNECTED;
    if (strcmp(s, "idle") == 0)                 return AgentState::IDLE;
    if (strcmp(s, "processing") == 0)           return AgentState::PROCESSING;
    if (strcmp(s, "awaiting_permission") == 0)  return AgentState::AWAITING_PERMISSION;
    if (strcmp(s, "awaiting_option") == 0)      return AgentState::AWAITING_OPTION;
    if (strcmp(s, "awaiting_diff") == 0)         return AgentState::AWAITING_DIFF;
    return AgentState::DISCONNECTED;
}
```

**解析**：守护进程推送的 `state` 字段是字符串（如 `"awaiting_permission"`），这里用一串 `strcmp` 映射到 6 态枚举。匹配守护进程的状态机：`idle`（待命）、`processing`（生成中）、`awaiting_permission`（YES/NO 权限）、`awaiting_option`（多选）、`awaiting_diff`（diff 审查）。未匹配默认 `DISCONNECTED`，保证未知状态不会卡住 UI。

UTF-8 安全的文本拷贝是个细节但关键的工程实践：

```cpp
// strncpy + NUL + drop any mid-UTF-8 cut. Daemon text (prompts, activity,
// timeline rows, 프로젝트명) can exceed these byte-sized buffers — a plain
// strncpy leaves a split 한글/CJK sequence that renders as a broken glyph.
static void copyTextU8(char* dst, size_t cap, const char* src) {
    if (cap == 0) return;
    strncpy(dst, src ? src : "", cap - 1);
    dst[cap - 1] = '\0';
    Utf8::utf8TrimEnd(dst);
}
```

**逐行解析**：
- 问题场景：守护进程发的韩文/CJK 文本可能超过定长缓冲（如 `projectName[40]` 字节）。`strncpy` 在字节中间截断，留下半个 UTF-8 序列（如 3 字节的 한 字被切掉 2 字节），渲染成乱码方块。
- `Utf8::utf8TrimEnd(dst)`（见 `util/utf8.h`）从末尾回退，丢弃不完整的首字节，确保结尾一定是完整字符。注释里特意写了 `프로젝트명`（韩文"项目名"）说明这是真实多语言场景踩过的坑。
- 这种"防御式国际化"在中文/韩文/日文项目里极常见，却常被忽略。

`handleStateUpdate` 处理代理状态推送：

```cpp
static void handleStateUpdate(JsonObject& obj) {
    lockState();
    g_state.state = parseState(obj["state"].as<const char*>());
    // Project & model
    if (obj["projectName"].is<const char*>())
        copyTextU8(g_state.projectName, sizeof(g_state.projectName), obj["projectName"].as<const char*>());
    if (obj["modelName"].is<const char*>())
        strncpy(g_state.modelName, obj["modelName"].as<const char*>(), sizeof(g_state.modelName) - 1);
    ...
    // Options array
    if (obj["options"].is<JsonArray>()) {
        JsonArray opts = obj["options"].as<JsonArray>();
        g_state.optionCount = min((int)opts.size(), 8);
        for (uint8_t i = 0; i < g_state.optionCount; i++) {
            JsonObject o = opts[i].as<JsonObject>();
            copyTextU8(g_state.options[i].label, sizeof(g_state.options[i].label), o["label"] | "");
            g_state.options[i].index = o["index"] | i;
            g_state.options[i].recommended = o["recommended"] | false;
            g_state.options[i].selected = o["selected"] | false;
            if (o["shortcut"].is<const char*>()) {
                strncpy(g_state.options[i].action, o["shortcut"].as<const char*>(),
                        sizeof(g_state.options[i].action) - 1);
            }
        }
    }
    ...
    // Mark that we've received real data from bridge
    g_state.dataReceived = true;
    // Derive creature states
    g_state.updateCreatureStates();
    unlockState();
}
```

**逐行解析**：
- 整个函数包在 `lockState()`/`unlockState()` 里——它一次性改写 `g_state` 的多个字段，必须原子化，否则 UI 核可能读到"半更新"状态（如新状态但旧选项）。
- `obj["x"].is<const char*>()` 先检查字段存在再取，避免 `nullptr` 传入 `strncpy`。
- 选项数组 `min((int)opts.size(), 8)`——硬上限 8 个选项，匹配 `PromptOption options[8]`。`o["index"] | i` 用 `|` 运算符提供默认值（ArduinoJson 特性，字段缺失时回退到 `i`）。
- 末尾 `updateCreatureStates()` 是派生：根据新 `AgentState` 推算章鱼该 SLEEPING/FLOATING/WORKING/ASKING，让 UI 直接读派生态而无需自己判断。

用量更新的"哨兵值"设计很值得学习：

```cpp
static void handleUsageUpdate(JsonObject& obj) {
    lockState();
    g_state.dataReceived = true;
    // Percent fields: use -1.0f sentinel for "no data" (0 is a valid value).
    // When bridge omits the field (stale TTL expired), clear to sentinel
    // instead of keeping the old sticky value.
    g_state.fiveHourPercent = obj["fiveHourPercent"].is<float>()
        ? obj["fiveHourPercent"].as<float>() : -1.0f;
    g_state.sevenDayPercent = obj["sevenDayPercent"].is<float>()
        ? obj["sevenDayPercent"].as<float>() : -1.0f;
```

**逐行解析**：
- **核心设计**：用 `-1.0f` 表示"无数据"，而不是 `0`。因为 0% 是合法值（刚重置配额），如果用 0 表示无数据，就无法区分"用光了"和"没数据"。
- 注释还指出"stale TTL 过期时清成哨兵而不是保留旧的粘性值"——避免显示过时的配额。
- `is<float>()` 检查字段是否为浮点类型，避免把字符串误转。

时间重置的相对化处理用了一个 lambda 把 ISO 8601 转成 "1h 23m"：

```cpp
    auto storeResetTime = [](JsonObject& obj, const char* key, char* out, size_t outLen) {
        if (!obj[key].is<const char*>()) { out[0] = '\0'; return; }
        const char* val = obj[key].as<const char*>();
        // Already formatted (no 'T' separator) — store directly
        if (strchr(val, 'T') == nullptr) {
            strncpy(out, val, outLen - 1);
            out[outLen - 1] = '\0';
            return;
        }
        // ISO 8601 — parse and compute relative time (needs NTP)
        struct tm tm = {};
        ...
        time_t resetEpoch = mktime(&tm);
        int offsetSec = (tzH * 3600 + tzM * 60) * (tzSign == '+' ? -1 : 1);
        resetEpoch += offsetSec;
        time_t now = time(nullptr);
        // Check if NTP has synced (time > 2025-01-01)
        if (now < 1735689600) {
            out[0] = '\0';
            return;
        }
        int diffSec = (int)(resetEpoch - now);
        ...
        if (diffMin < 60) {
            snprintf(out, outLen, "%dm", diffMin);
        } else {
            int h = diffMin / 60;
            int m = diffMin % 60;
            if (h < 24) {
                if (m > 0) snprintf(out, outLen, "%dh %dm", h, m);
                else snprintf(out, outLen, "%dh", h);
            } else {
                int d = h / 24;
                int rh = h % 24;
                if (rh > 0) snprintf(out, outLen, "%dd %dh", d, rh);
                else snprintf(out, outLen, "%dd", d);
            }
        }
    };
```

**逐行解析**：
- 双格式兼容：串行桥接发的是预格式化的 `"1h 23m"`（无 `T`），直接存；WebSocket 发的是 ISO 8601（`2026-07-20T15:30:00+09:00`，有 `T`），需要解析+算相对时间。
- 时区处理：`sscanf` 解析 `+09:00` 偏移，乘以 `-1`（因为要转 UTC，东半区要减）。`mktime` 用本地时区（但固件 `configTzTime("UTC")` 把本地设成了 UTC，所以等价于 `timegm`）。
- **NTP 检查**：`now < 1735689600`（即 2025-01-01 之前的 epoch）意味着 NTP 还没同步，`time(nullptr)` 返回的是编译期默认值，算出的相对时间无意义，于是清空返回。这种"等时间准了再算"的防御避免了开机瞬间显示错误的倒计时。
- 分档格式化：`<60m` 显示分，`<24h` 显示 `时 分`，`>=24h` 显示 `天 时`。

### 4. WiFi 配网与射频停泊（wifi_manager.cpp）

`wifiInit` 用 WiFiManager 实现非阻塞配网：

```cpp
namespace Net {
void wifiInit() {
    WiFi.mode(WIFI_STA);
    // Disable modem power-save: with PS on, classic ESP32 WiFi stalls periodically
    // (TCP connect timeouts, resets, WS drops within seconds of connecting).
    WiFi.setSleep(false);
    // Non-blocking portal mode: if no saved credentials, starts AP
    // but returns immediately so serial can still work
    wm.setConfigPortalBlocking(false);
    wm.setConnectTimeout(8);
    wm.setConfigPortalTimeout(0);  // Portal stays open until configured
    wm.setTitle("AgentDeck");
```

**逐行解析**：
- `WiFi.setSleep(false)` 关闭调制解调器省电——注释明确说明这是踩坑后的结论：开省电会周期性卡顿，导致 TCP 连接超时、WS 几秒内掉线。这是 ESP32 经典 WiFi 稳定性问题的标准对策。
- `setConfigPortalBlocking(false)` 让门户非阻塞：没凭据时起 AP（`AgentDeck-Setup`）但立即返回，不卡住串行主循环。这是固件能"USB 永远可用"的前提。
- `setConfigPortalTimeout(0)` 门户常开直到配好——适合开发板这种"配一次就长期用"的场景。

连接成功后启动 NTP：

```cpp
    if (wm.autoConnect(AP_SSID)) {
        IPAddress ip = WiFi.localIP();
        snprintf(ipBuf, sizeof(ipBuf), "%d.%d.%d.%d", ip[0], ip[1], ip[2], ip[3]);
        Serial.printf("[WiFi] Connected: %s\n", ipBuf);
        // Sync NTP so time(nullptr) works for reset time parsing
        configTzTime("UTC", "pool.ntp.org", "time.google.com");
        Serial.println("[WiFi] NTP sync started (UTC)");
        wifiWasConnected = true;
        portalActive = false;
    } else {
        Serial.printf("[WiFi] No saved credentials — AP portal active: %s\n", AP_SSID);
        Serial.println("[WiFi] Connect to AP and visit 192.168.4.1 to configure");
        portalActive = true;
    }
}
```

**逐行解析**：
- `autoConnect(AP_SSID)` 用保存的凭据连，失败则起 `AgentDeck-Setup` AP。
- `configTzTime("UTC", ...)` 设时区为 UTC 并指定两个 NTP 服务器（`pool.ntp.org` 主、`time.google.com` 备）。这正是上文 `protocol.cpp` 里 `mktime` 能正确算相对时间的基础——本地时区被设成 UTC，`mktime` 等价于 `timegm`。
- 没凭据时打印提示，让用户连 AP 后访问 `192.168.4.1` 配网（标准 captive portal 流程）。

**射频停泊**（radio parking）是这套固件最巧妙的设计：

```cpp
void wifiSetRadioParked(bool parked) {
    if (parked) {
        radioParked = true;
        wifiWasConnected = false;
        WiFi.mode(WIFI_OFF);
    } else {
        radioParked = false;
        WiFi.mode(WIFI_STA);
        WiFi.setSleep(false);
        WiFi.reconnect();
    }
}
```

**解析**：`wifiSetRadioParked(true)` 把 WiFi 关掉（`WIFI_OFF`），`false` 恢复。结合 `main.cpp` 里的逻辑——当串行稳定超过 4 秒，板子认定 USB 是主传输，关掉无线省 2.4GHz 带宽（给其他 WiFi-only 板让路）；串行断了立刻恢复无线。这种"插 USB 就静默无线"的策略让多板部署不互相抢信道。

IPS10（ESP32-P4 + C6 双芯）有特例——不能 `WIFI_OFF`，否则会触发 SDIO 断言：

```cpp
        // ESP32-P4 reaches WiFi through an ESP32-C6 running ESP-Hosted over
        // SDIO. WiFi.mode(WIFI_OFF) tears that transport down all the way
        // through hostedDeinitWiFi(). If a final RX packet is still in flight,
        // the C6 can enqueue into buffers that the P4 just destroyed and trip
        // sdio_rx_get_buffer/sdio_push_data_to_queue assertions.
        // ...Disassociate the STA while leaving ESP-Hosted initialized.
        if (WiFi.getMode() != WIFI_OFF && WiFi.isConnected()) {
            if (!WiFi.disconnect(false, 1000)) {
                Serial.println("[WiFi] IPS10 STA quiesce timed out; hosted transport kept alive");
            }
        }
```

**逐行解析**：
- 这段注释是真实硬件调试的结晶：ESP32-P4 没有原生 WiFi，要通过 C6 协处理器经 SDIO 访问。`WIFI_OFF` 会把整个 ESP-Hosted 传输层拆掉，但若此时还有在途 RX 包，C6 会往已销毁的缓冲入队，触发 `sdio_rx_get_buffer`/`sdio_push_data_to_queue` 断言崩溃。
- 对策：只 `disconnect` STA（解除关联），保留 ESP-Hosted 初始化状态，让链路安静但不拆除。`disconnect(false, 1000)` 的 `false` 表示不擦除配置，`1000` 是 1s 超时。
- 这种"同功能不同实现"的板级适配，体现了固件对多硬件的工程严谨。

### 5. mDNS 发现与守护端点优先（mdns_discovery.cpp）

`mdnsPoll` 负责发现局域网内的 AgentDeck 桥接：

```cpp
namespace Net {
void mdnsInit() {
    if (!MDNS.begin("agentdeck-display")) {
        Serial.println("[mDNS] Failed to start");
        return;
    }
    Serial.println("[mDNS] Started, browsing for _agentdeck._tcp");
    memset(&discovered, 0, sizeof(discovered));
}
bool mdnsPoll(BridgeInfo& out) {
    uint32_t now = millis();
    if (now - lastQueryMs < QUERY_INTERVAL_MS) {
        if (hasNew) {
            out = discovered;
            hasNew = false;
            return true;
        }
        return false;
    }
    lastQueryMs = now;
    int n = MDNS.queryService("_agentdeck", "_tcp");
    if (n <= 0) return false;
```

**逐行解析**：
- `MDNS.begin("agentdeck-display")` 给板子起 mDNS 主机名（`agentdeck-display.local`）。
- `QUERY_INTERVAL_MS = 5000` 节流：5 秒内不重复查询（配合 `main.cpp` 里"只在未连接时 poll"进一步降低开销）。
- `hasNew` 标志：查询到新结果后保留，下次 poll（即使未到间隔）也能返回，避免漏掉刚发现的结果。

发现多个端点时的优先级选择：

```cpp
    // Prefer daemon bridge for consistent state (daemon aggregates all sessions)
    int daemonIdx = -1;
    int firstIdx = -1;
    for (int i = 0; i < n; i++) {
        uint16_t port = MDNS.port(i);
        if (port == 0) continue;
        if (firstIdx < 0) firstIdx = i;
        // Check agent TXT record for daemon type
        int numKeys = MDNS.numTxt(i);
        for (int k = 0; k < numKeys; k++) {
            if (MDNS.txtKey(i, k) == "agent" && MDNS.txt(i, k) == "daemon") {
                daemonIdx = i;
                break;
            }
        }
        if (daemonIdx >= 0) break;
    }
    int selected = (daemonIdx >= 0) ? daemonIdx : firstIdx;
```

**逐行解析**：
- **守护进程优先**：局域网里可能同时有会话桥（端口 9121+）和守护进程（9120）。守护进程聚合所有会话状态，是"全局视图"来源；会话桥只看单个会话。所以遍历 TXT 记录找 `agent=daemon`，优先选它。
- 找不到守护进程才回退到第一个（`firstIdx`）。
- `port == 0 continue` 跳过无效条目。

TXT 记录解析提取 token 和元数据：

```cpp
        // Parse TXT records
        int numKeys = MDNS.numTxt(selected);
        for (int k = 0; k < numKeys; k++) {
            String key = MDNS.txtKey(selected, k);
            String val = MDNS.txt(selected, k);
            if (key == "token") {
                strncpy(discovered.token, val.c_str(), sizeof(discovered.token) - 1);
            } else if (key == "project") {
                strncpy(discovered.project, val.c_str(), sizeof(discovered.project) - 1);
            } else if (key == "agent") {
                strncpy(discovered.agent, val.c_str(), sizeof(discovered.agent) - 1);
            }
        }
        Serial.printf("[mDNS] Found bridge: %s:%d agent=%s project=%s\n",
                       discovered.ip, discovered.port, discovered.agent, discovered.project);
        hasNew = true;
        out = discovered;
        return true;
```

**逐行解析**：
- 桥接在 mDNS 广播里塞了 `token`（认证）、`project`（项目名）、`agent`（类型）三条 TXT 记录。
- **token 经 mDNS 分发**：这样板子发现桥接时直接拿到 token，无需用户手动输入，实现"开机自动连"。当然这也依赖局域网可信（README 提到本地连接免认证、远程需 token）。
- 解析后存入 `discovered`（`BridgeInfo` 结构），供 `main.cpp` 拼接成 `wsConnect(ip, port, token)`。

### 6. 状态机与生物态派生（agent_state.h）

`DashboardState` 是固件的数据中枢，结构定义体现"显式状态机 + 派生"的设计：

```cpp
enum class AgentState : uint8_t {
    DISCONNECTED = 0,
    IDLE,
    PROCESSING,
    AWAITING_PERMISSION,
    AWAITING_OPTION,
    AWAITING_DIFF
};
enum class CreatureState : uint8_t {
    SLEEPING = 0,
    FLOATING,
    WORKING,
    ASKING
};
```

**解析**：`enum class`（强类型枚举）防止隐式转 int，`uint8_t` 底层省内存。6 个代理状态对应 4 个生物态——这是"把抽象状态可视化为具象生物"的核心映射。

派生逻辑在 `updateCreatureStates`：

```cpp
    void updateCreatureStates() {
        switch (state) {
            case AgentState::DISCONNECTED:
                creatureState = CreatureState::SLEEPING;
                tetraState = TetraState::HOVERING;
                break;
            case AgentState::IDLE:
                creatureState = CreatureState::FLOATING;
                tetraState = TetraState::CIRCLING;
                break;
            case AgentState::PROCESSING:
                creatureState = CreatureState::WORKING;
                tetraState = TetraState::STREAMING;
                break;
            case AgentState::AWAITING_PERMISSION:
            case AgentState::AWAITING_OPTION:
            case AgentState::AWAITING_DIFF:
                creatureState = CreatureState::ASKING;
                tetraState = TetraState::HOVERING;
                break;
        }
        // Derive crayfish state from gateway when no sessions_list received.
        if (crayfishCount == 0) {
            if (gatewayHasError) {
                crayfishState = CrayfishState::SICK;
            } else if (gatewayConnected) {
                crayfishState = CrayfishState::SITTING;
            } else {
                crayfishState = CrayfishState::DORMANT;
            }
        }
    }
```

**逐行解析**：
- 三种 `awaiting_*` 状态都映射到 `ASKING`（章鱼冒问号气泡），简化 UI 渲染。
- 小龙虾（crayfish）代表 OpenClaw Gateway：网关出错就 `SICK`（生病），已认证就 `SITTING`（坐着），否则 `DORMANT`（休眠）。`crayfishCount == 0` 表示还没收到会话列表，从网关状态派生；一旦收到会话列表，`handleSessionsList` 会按实际会话数覆盖。
- 这种"单一真相源派生"避免了 UI 核自己判断状态，减少不一致 bug。

断开时的状态清理保留连接记忆：

```cpp
    void markBridgeDisconnected() {
        wsConnected = false;
        state = AgentState::DISCONNECTED;
        projectName[0] = '\0';
        ...
        optionCount = 0;
        clearSessions();
        gatewayAvailable = false;
        gatewayConnected = false;
        gatewayHasError = false;
        dataReceived = false;
        fiveHourPercent = -1.0f;
        sevenDayPercent = -1.0f;
        ...
        usageStale = true;
        updateCreatureStates();
    }
```

**逐行解析**：
- 清空所有"易失"代理数据（项目名、模型、选项、会话、用量），但**保留 `lastMessageMs` 等连接记忆**（注释在结构定义处强调）。
- 用量字段清成 `-1.0f` 哨兵而非 `0`，UI 据此隐藏配额条而不是显示 0%。
- 末尾 `updateCreatureStates()` 把章鱼切到 `SLEEPING`，UI 立刻反映断连。

### 7. 板级配置常量（config.h）

`config.h` 用编译宏为不同板子裁剪资源上限，是内存优化的关键：

```cpp
// ===== Terrarium =====
#if defined(BOARD_LED8X32)
constexpr uint8_t  MAX_OCTOPUS         = 1;
constexpr uint8_t  MAX_CLOUD           = 0;
...
constexpr uint8_t  WAVE_SEGMENTS       = 0;
#elif defined(BOARD_TTGO) || defined(BOARD_ESP32_C6_147)
constexpr uint8_t  MAX_OCTOPUS         = 2;
constexpr uint8_t  MAX_CLOUD           = 1;
constexpr uint8_t  MAX_BUBBLES         = 3;   // trimmed (was 6) — less constant motion
...
#elif defined(BOARD_IPS10)
constexpr uint8_t  MAX_OCTOPUS         = 8;
constexpr uint8_t  MAX_CLOUD           = 6;
constexpr uint8_t  MAX_BUBBLES         = 30;
...
#else
constexpr uint8_t  MAX_OCTOPUS         = 6;
...
#endif
```

**逐行解析**：
- 同一套动画引擎，按板子能力分四档：TC001（8×32 LED）只能 1 个章鱼、0 气泡；TTGO/C6 小屏 2 章鱼 + 3 气泡（注释"原 6，减到 3 少点恒定运动"省 CPU）；IPS10 大屏 8 章鱼 + 30 气泡；默认 6 章鱼。
- `constexpr` 编译期求值，不占 RAM。这种"同源代码不同资源上限"让一份固件支持 8 块板而内存占用各异。

消息大小上限也按板分级：

```cpp
#if defined(BOARD_TTGO) || defined(BOARD_ESP32_C6_147) || defined(BOARD_LED8X32)
constexpr size_t PROTOCOL_MAX_MSG_BYTES = 8192;
#else
constexpr size_t PROTOCOL_MAX_MSG_BYTES = 65536;
#endif
```

**解析**：无 PSRAM 的小板（TTGO/C6/TC001）把入站帧上限压到 8KB，防止超大的 `sessions_list`/`timeline_history` 把 `JsonDocument` 撑爆导致堆碎片化；有 PSRAM 的大板放宽到 64KB。这是"按硬件能力分级容错"的典范。

## API 使用指南

### Python：通过守护进程 HTTP API 查询状态

AgentDeck 守护进程在 `0.0.0.0:9120` 暴露 HTTP 接口，本地免认证、远程需 token。

```python
import requests

DAEMON = "http://127.0.0.1:9120"
TOKEN  = ""  # 本地连接留空；远程填 ~/.agentdeck/auth-token 里的值

headers = {"Authorization": f"Bearer {TOKEN}"} if TOKEN else {}

# 1. 健康检查
r = requests.get(f"{DAEMON}/health", headers=headers, timeout=3)
print("daemon:", r.json())

# 2. 列出已连接设备（含 ESP32 板）
r = requests.get(f"{DAEMON}/devices", headers=headers, timeout=3)
for d in r.json().get("devices", []):
    print(d.get("clientType"), d.get("board"), d.get("ip"))

# 3. 列出当前会话
r = requests.get(f"{DAEMON}/sessions", headers=headers, timeout=3)
for s in r.json().get("sessions", []):
    print(s["agentType"], s["projectName"], s["state"])

# 4. 向某会话发送指令（approve / interrupt / select_option）
requests.post(f"{DAEMON}/command", headers=headers, json={
    "sessionId": "<session-id>",
    "type": "respond",
    "value": "yes"
}, timeout=3)
```

### cURL：手动控制 ESP32 板

通过 `agentdeck` CLI 触发 ESP32 相关操作（CLI 会调用守护进程）：

```bash
# 查看所有已连接设备（含 ESP32 板的 IP/板型）
curl -s http://127.0.0.1:9120/devices | python3 -m json.tool

# 通过串行向 ESP32 板请求 device_info
curl -s -X POST http://127.0.0.1:9120/esp32/device_info \
  -H "Content-Type: application/json" -d '{"target":"round_amoled"}'

# 远程推送 OTA 固件到 ESP32 板（先编译再烧录）
agentdeck esp32-ota round_amoled --build

# WiFi 配网（USB 串行连接板子时）
agentdeck wifi-setup

# 生成配对二维码（供 Android/Apple 扫码连守护进程）
agentdeck qr
```

### WebSocket：直接订阅状态流

ESP32 板与守护进程走的是 `ws://<daemon-ip>:9120/?token=<token>&clientType=esp32`，可用 `websocat` 模拟：

```bash
# 订阅状态更新（会持续收到 state_update / usage_update / sessions_list JSON）
TOKEN=$(cat ~/.agentdeck/auth-token)
websocat "ws://127.0.0.1:9120/?token=$TOKEN&clientType=esp32"

# 主动发送一个 query_usage 命令
echo '{"type":"query_usage"}' | websocat -1 "ws://127.0.0.1:9120/?token=$TOKEN&clientType=esp32"
```

## 编译与部署

### 环境搭建

| 组件 | 版本 | 说明 |
|------|------|------|
| PlatformIO Core | ≥ 6 | `pip install platformio` 或 VSCode 插件 |
| Arduino-ESP32 | v3.x（ESP-IDF 5.x） | `platformio.ini` 指定 `platform = espressif32` |
| 框架 | Arduino | 主框架，配 FreeRTOS、ESPmDNS、WiFi、Update |
| 依赖库 | arduinoWebSockets、ArduinoJson、Adafruit GFX、LVGL、FastLED、GxEPD2、WiFiManager | 见 `platformio.ini` 的 `lib_deps` |
| Python | ≥ 3.8 | 跑 `scripts/git_rev.py` 注入构建哈希 |

### 编译配置表（platformio.ini 节选）

每块板一个 `[env:xxx]`，通过 `build_flags` 选择板级宏：

| Env | 板子 | 关键 build_flags | 分区表 |
|-----|------|------------------|--------|
| `round_amoled` | Round AMOLED 1.8" | `-DBOARD_AMOLED -DIS_ROUND` | `round_amoled.csv` |
| `ips35` | IPS LCD 3.5" | `-DBOARD_IPS35` | `ips_35.csv` |
| `86box` | B86 Box 4" | `-DBOARD_86_BOX` | `box_86_ota.csv` |
| `ttgo_t_display` | TTGO 1.14" | `-DBOARD_TTGO` | `ttgo_t_display.csv` |
| `esp32_c6_147` | Waveshare 1.47" | `-DBOARD_ESP32_C6_147` | `jc8012p4a1c.csv` |
| `ips10` | IPS 10.1" | `-DBOARD_IPS10` | `jc8012p4a1c_ota.csv` |
| `ulanzi_tc001` | TC001 LED 矩阵 | `-DBOARD_LED8X32` | `ulanzi_tc001.csv` |
| `inkdeck` | InkDeck e-ink | `-DBOARD_INKDECK` | — |

### 烧录步骤

**方式一：PlatformIO 本地编译烧录**

```bash
cd esp32
# 注入 git 修订哈希（写入 config.h 的 GIT_SHA / BUILD_EPOCH）
python scripts/git_rev.py
# 编译指定板子（以 Round AMOLED 为例）
pio run -e round_amoled -t upload      # USB 连板子
# 监视串口（看 [Net]/[UI]/[WS]/[mDNS] 日志）
pio device monitor -e round_amoled
```

**方式二：守护进程远程 OTA（WiFi 板）**

```bash
# 先 USB 烧一次 bootloader + 初始固件（首次必须）
pio run -e round_amoled -t upload

# 之后无需 USB，远程推送
agentdeck esp32-ota round_amoled --build   # 编译并 OTA 推送
# 或指定本地固件路径
agentdeck esp32-ota round_amoled --firmware .pio/build/round_amoled/firmware.bin
```

**方式三：WiFi 配网（首次上电）**

```bash
# USB 串行连板子，运行配网向导
agentdeck wifi-setup
# 或手动：手机连 AgentDeck-Setup AP → 浏览器访问 192.168.4.1 → 选 WiFi 输密码
```

### 验证

烧录后串口应出现：

```
[Net] Task started on core 0
[WiFi] Connected: 192.168.1.42
[WiFi] NTP sync started (UTC)
[mDNS] Started, browsing for _agentdeck._tcp
[mDNS] Found bridge: 192.168.1.10:9120 agent=daemon project=my-app
[WS] Connecting to 192.168.1.10:9120
[WS] Connected to 192.168.1.10:9120
[UI] Task started on core 1
[UI] Screens created, entering main loop
```

屏幕上应显示 Splash → Aquarium 水族箱，章鱼根据代理状态切换 FLOATING/WORKING/ASKING。

## 项目亮点与适用场景

### 项目亮点

1. **双核隔离的工程范式**：网络核与 UI 核严格分离，跨核通信用互斥量 + 环形出队队列，既保证线程安全（arduinoWebSockets 非线程安全）又不阻塞 UI。这是 ESP32 多任务架构的优秀范本。
2. **传输自愈与端点自愈**：mDNS 发现 + NVS 持久化 + 指数退避 + 长断连强制 mDNS 刷新，让板子在守护进程搬家、DHCP 漂移、WiFi 抖动等各种场景下都能自恢复，注释里点名修复过的真实 bug（如 `67934f94`）。
3. **串行优先 + 射频停泊**：插 USB 时自动关 WiFi 省 2.4GHz 带宽，给多板部署让路；IPS10 双芯板的 SDIO 断言问题用"只断 STA 不拆传输层"优雅绕过，体现深度硬件理解。
4. **生物态可视化**：把抽象的代理状态机映射成章鱼/小龙虾/霓虹灯鱼的水族箱动画，让"AI 在干嘛"一眼可见——既是工程也是设计语言。
5. **板级裁剪一份代码八板用**：`config.h` 用编译宏分档资源上限与消息大小，让从 256 像素 LED 到 10.1" MIPI-DSI 大屏共用同一套动画引擎，内存占用各得其所。
6. **国际化细节**：`copyTextU8` 处理韩文/CJK 截断，NTP 未同步时跳过相对时间计算，这些"小处"决定了多语言项目的可用性。

### 适用场景

- **AI 编码工作流的物理驾驶舱**：用 Stream Deck+/ESP32 屏/D200H 等实体按键和屏幕驾驶 Claude Code/Codex，比切回终端更顺手。
- **多代理并行的全局仪表盘**：一台机器跑多个 AI 代理时，用 ESP32 大屏做常亮"鱼缸监控"，哪个代理在等权限、哪个在跑工具一目了然。
- **ESP32 FreeRTOS + WebSocket 学习参考**：双核任务、互斥量、WebSocket 退避、mDNS 发现、ArduinoJson 协议解析、WiFiManager 配网、OTA，覆盖了 IoT 固件几乎所有典型模块。
- **多板 HAL 与 OTA 实践**：同一份代码适配 8 块板 + 双 OTA 分区远程升级，是硬件产品化固件的范例。
- **唤醒词 + 本地 AI 推理**：ESP32 端 microWakeWord（TFLite）做离线唤醒，配合宿主机 LLM 实现全离线语音助手。

## 总结

AgentDeck 的 ESP32 固件端是一套"把 AI 代理状态变成可触碰物理界面"的完整工程。它的价值不在于单个炫酷功能，而在于把 **FreeRTOS 双核架构、传输冗余自愈、mDNS 零配置发现、跨核线程安全、板级资源裁剪、多语言 UTF-8 安全、OTA 升级**这些嵌入式工程的标准课题，在一个真实迭代、有真实 bug 注释、覆盖 8 块板的项目里打磨成了一套可读、可维护、可扩展的固件。

对嵌入式开发者而言，它是"如何用 ESP32 做一个连得上、连得稳、断得了、还能自己恢复的联网显示终端"的极佳教材——尤其是它对 2.4GHz 带宽、WiFi 省电坑、ESP-Hosted SDIO 断言这些真实硬件陷阱的处理，远比教科书上"连 WiFi 收 JSON"的简化模型来得扎实。结合其 AI 代理主题与生物态可视化的设计语言，这个项目也展示了嵌入式不再只是"传感+控制"，而正在成为人与 AI 协作的新交互层。

---

📝 作者：蔡浩宇（jun-chy） / 📅 日期：2026-07-20 / 🔗 项目地址：[puritysb/AgentDeck](https://github.com/puritysb/AgentDeck)
