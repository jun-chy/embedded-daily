# ESP32 Bit Pirate 深度解析：把两美元开发板变成全协议硬件分析利器

> 📅 日期：2026-07-23 ｜ 🏷️ 分类：ESP32-S3 / 硬件安全工具 ｜ ⭐ Stars：4360 ｜ 👤 作者：蔡浩宇（jun-chy）

---

## 项目链接

| 属性 | 信息 |
|------|------|
| GitHub 地址 | [geo-tp/ESP32-Bit-Pirate](https://github.com/geo-tp/ESP32-Bit-Pirate) |
| 作者 | geo-tp |
| 许可证 | MIT |
| 最近更新 | 2026-07-23（当日活跃） |
| 主要语言 | C++ |
| 默认分支 | pioarduino |
| 官方网站 | [geo-tp.github.io/ESP32-Bit-Pirate](https://geo-tp.github.io/ESP32-Bit-Pirate/) |

ESP32 Bit Pirate 是一款开源固件，受传奇的 Bus Pirate 启发，将 ESP32-S3 开发板变成一台多协议硬件分析工具。它通过串口终端或浏览器 Web CLI 嗅探、发送、脚本化地与 I2C、UART、1-Wire、SPI、CAN、JTAG 等数字协议交互，还能操作蓝牙、Wi-Fi、Sub-GHz 射频和 RFID 等无线协议。项目当前已支持 23 种工作模式、10 余款主流开发板，star 数突破 4360，是 GitHub 上增长最快的嵌入式安全工具之一。

---

## 核心特性一览

| 特性 | 说明 |
|------|------|
| 多协议数字总线 | I2C / SPI / UART / HDUART / 1-Wire / 2-Wire / 3-Wire / CAN / JTAG / I2S |
| 射频与无线 | Sub-GHz（CC1101）/ LoRa（SX1262）/ RF24 / Bluetooth BLE / Wi-Fi / Ethernet / FM 广播 / RFID（PN532）/ 蜂窝 SIM |
| 三种 CLI 界面 | USB 串口 / Wi-Fi 浏览器 Web CLI / Cardputer 独立模式（板载键盘+屏幕） |
| 协议嗅探器 | I2C、UART、SPI、1-Wire、2-Wire、CAN、Wi-Fi、蓝牙、Sub-GHz |
| 脚本引擎 | Bus Pirate 风格字节码指令 + Python 串口自动化 |
| 逻辑分析仪 | PinAnalyzer 引脚电平采样、脉冲环缓冲、突发检测、PWM/伺服周期提取 |
| 存储转储 | I2C/SPI/3-Wire EEPROM 读写转储、SPI Flash 编程、SD 卡 |
| 红外遥控 | 80+ 红外协议、万能遥控、Device-B-Gone |
| LED 控制 | 近 50 种可寻址 LED 协议 |
| Web 烧录器 | 浏览器直接刷固件，无需安装工具 |

---

## 硬件架构

### 系统组成图

```
                    ┌─────────────────────────────────────────────┐
                    │           ESP32-S3 (Xtensa LX7 双核)         │
                    │           16MB Flash / 2-8MB PSRAM           │
                    │                                             │
   USB / 串口 <────►│  HostSerial ──► SerialTerminalView/Input     │
   (CP2102/CDC)     │                                             │
                    │  Wi-Fi STA/AP ──► HttpServer + WebSocket     │
   浏览器 <────────►│  (192.168.x.x)    │                          │
                    │                   ▼                          │
                    │          ActionDispatcher (中央分发)          │
                    │         ╱   │   │   │   │   ╲                 │
                    │       I2C  UART SPI 1WIRE SUBGHZ ...RFID      │
                    └──┬────┬────┬───┬───┬────┬────┬────┬───────────┘
                       │    │    │   │   │    │    │    │
                    GPIO组  SDA  MOSI DQ  CC1101 SX1262 PN532 IR
                    (可重映射引脚映射, 通过 platformio.ini -D 宏定义)
```

### 引脚配置（以 LILYGO T-Display S3 为例）

该配置摘自 `platformio.ini` 中 `[env:t-display-s3]` 段，展示了每条协议总线的引脚映射。所有引脚通过编译宏定义，换板只需改配置无需改代码：

| 协议 | 关键引脚 | 说明 |
|------|----------|------|
| OneWire | DQ=GPIO16 | 1-Wire 数据线 |
| UART | RX=GPIO44 / TX=GPIO43 | 波特率 9600 |
| HDUART | GPIO44 | 半双工单线 UART |
| I2C | SCL=GPIO17 / SDA=GPIO16 | 100kHz |
| SPI | CS=GPIO10 / CLK=GPIO12 / MISO=GPIO13 / MOSI=GPIO11 | |
| LoRa SX1262 | SCK=12 / MISO=13 / MOSI=11 / CS=10 / RST=3 / BUSY=2 / DIO1=1 | |
| SubGHz CC1101 | CS=10 / SCK=12 / SI=13 / SO=11 / GDO=3 | |
| RF24 | CSN=10 / CE=3 / SCK=12 / MISO=11 / MOSI=13 | |
| CAN | CS=10 / SCK=12 / SI=13 / SO=11 | 125 kbps |
| Ethernet W5500 | CS=10 / CLK=12 / MISO=13 / MOSI=11 / IRQ=3 | |
| 红外 | TX=GPIO43 / RX=GPIO44 | |
| JTAG 扫描 | 3,10,11,12,13,16,17 | 候选扫描引脚列表 |
| 板载 LED | GPIO42（RGB） | |

### 支持设备清单

| 设备 | 特色资源 |
|------|----------|
| ESP32-S3 Dev Kit | 20+ GPIO，1 按钮 |
| LILYGO T-Display S3 | 13 GPIO、屏幕、2 按钮 |
| LILYGO T-Embed / T-Embed CC1101 | 屏幕、编码器、扬声器、麦克风、SD 卡、CC1101、PN532 |
| M5 Cardputer | 屏幕、键盘、麦克风、扬声器、IR、SD、独立模式 |
| M5 StampS3 / StickS3 / AtomS3 | 紧凑型，9-13 GPIO |
| Seeed XIAO S3 | 9 GPIO，超小尺寸 |
| Heltec WiFi LoRa 32 V3/V4 | 板载 SX1262 LoRa |
| Heltec Vision Master T190 | 大屏 + 板载 SX1262 |

任何具备 8MB Flash 的 ESP32-S3 板均可刷入 S3 DevKit 固件运行。

---

## 固件架构

### 目录结构与模块职责

项目采用清晰的分层架构，源码位于 `src/` 下 22 个职责分明的子目录，外加 `lib/` 中的协议驱动库：

| 目录 | 职责 | 代表文件 |
|------|------|----------|
| `src/Boards/` | 板级抽象，每款板一个类，暴露 DeviceView/Input/HostSerial | CardputerBoard, TDisplayS3Board |
| `src/Configurators/` | 启动配置流程：终端类型、Wi-Fi 模式、启动模式 | TerminalTypeConfigurator |
| `src/Dispatchers/` | 中央命令分发器，解析并路由用户输入 | ActionDispatcher |
| `src/Controllers/` | 23 个协议控制器，每个模式一个，解析命令并调用 Service | I2cController, SubGhzController |
| `src/Services/` | 硬件访问层，直接操作外设 | （通过接口被 Controller 调用） |
| `src/Servers/` | HTTP / WebSocket / DNS 服务器 | WebSocketServer, HttpServer, DnsServer |
| `src/Analyzers/` | 信号分析仪：引脚逻辑、二进制、SubGHz | PinAnalyzer |
| `src/Shells/` | 21 个协议专属 Shell，提供交互式子命令 | I2cEepromShell, SpiFlashShell |
| `src/Managers/` | 命令行管理：别名、历史、行缓冲、用户输入 | CommandLineManager |
| `src/Transformers/` | 命令/指令转换器：宏、管道、字节码 | InstructionTransformer |
| `src/Interfaces/` | 40+ 纯虚接口，实现依赖反转 | II2cService, ISubGhzService |
| `src/Providers/` | 依赖注入容器，按配置装配所有组件 | DependencyProvider |
| `src/Views/` | 终端视图：串口 / Web / Cardputer 屏 | SerialTerminalView |
| `src/Models/` | 数据模型：TerminalCommand、ByteCode | TerminalCommand |
| `src/Enums/` | 枚举：模式、终端类型、字节码 | ModeEnum |
| `lib/` | 第三方协议驱动 | PN532, SX126x-Arduino, CC1101, RF24, IRremote |

命令在系统中的流转路径，`main.cpp` 顶部注释给出了精炼的描述：

```
用户输入命令 → TerminalInput（读字符）→ ActionDispatcher（路由）
→ Controller（解析校验）→ Service（访问硬件/协议）→ Controller（处理响应）
→ TerminalView（显示输出）
```

---

## 核心代码深度分析

以下所有代码片段均从项目真实源码提取，逐段解析其设计意图与实现细节。

### 1. 启动流程与板级抽象（main.cpp）

`main.cpp` 是整个固件的入口，负责板级初始化、终端模式选择和分发器启动。它通过编译时宏 `#ifdef` 选择具体板卡，再通过配置器选择终端类型，最后构建依赖注入容器并进入永久循环。

```cpp
void setup() {
    #if defined(DEVICE_STICKS3)
        StickS3Board board;
        board.initialize();
        IDeviceView& deviceView = board.getDeviceView();
        IInput& deviceInput = board.getDeviceInput();
        IHostSerial& hostSerial = board.getHostSerial();
    #elif defined(DEVICE_CARDPUTER)
        CardputerBoard board;
        board.initialize();
        IDeviceView& deviceView = board.getDeviceView();
        IInput& deviceInput = board.getDeviceInput();
        IHostSerial& hostSerial = board.getHostSerial();
    // ... 其余板卡以相同模式处理 ...
    #else
        BoardHostSerial fallbackHostSerial;
        NoScreenDeviceView fallbackDeviceView;
        DefaultInput fallbackDeviceInput;
        fallbackDeviceView.initialize();
        IDeviceView& deviceView = fallbackDeviceView;
        IInput& deviceInput = fallbackDeviceInput;
        IHostSerial& hostSerial = fallbackHostSerial;
    #endif
```

这段代码的核心设计是**板级多态抽象**。每款板卡（StickS3、Cardputer、TDisplayS3 等）都被封装为一个 Board 类，对外统一暴露三个接口引用：`IDeviceView`（屏幕显示）、`IInput`（物理按键）和 `IHostSerial`（主机串口链路）。编译时通过 `-DDEVICE_xxx` 宏选定具体实现，运行时上层代码只持有接口引用，完全不感知具体硬件。当没有任何板卡宏匹配时，`#else` 分支提供无屏幕 + 默认输入 + 串口的后备方案，确保固件在任何 ESP32-S3 板上都能启动。

接下来是终端类型选择和 Wi-Fi 配置：

```cpp
    NvsService bootNvsService;
    BootModeConfigurator bootModeConfigurator(deviceView, deviceInput, bootNvsService, hostSerial);
    if (bootModeConfigurator.configure()) {
        return;
    }

    LittleFsService littleFsService;
    UtilityService utilityService;
    GlobalState& state = GlobalState::getInstance();

    HorizontalSelector selector(deviceView, deviceInput, utilityService);
    TerminalTypeConfigurator configurator(selector);
    TerminalTypeEnum terminalType = configurator.configure();
    std::string webIp = "0.0.0.0";

    if (terminalType == TerminalTypeEnum::WiFiClient || terminalType == TerminalTypeEnum::WiFiAp) {
        WifiTypeConfigurator wifiTypeConfigurator(deviceView, deviceInput, utilityService);
        webIp = wifiTypeConfigurator.configure(terminalType);
        if (webIp == "0.0.0.0") {
            terminalType = TerminalTypeEnum::SerialPort;
        } else {
            state.setTerminalIp(webIp);
        }
    }
    state.setTerminalMode(terminalType);
```

`BootModeConfigurator` 先检查 NVS 中是否设置了 USB 适配器启动模式，若配置成功则直接返回（进入 USB 适配器角色）。随后 `TerminalTypeConfigurator` 通过 `HorizontalSelector` 让用户在串口、Wi-Fi 客户端、Wi-Fi 热点、独立模式间选择。如果选择了 Wi-Fi 模式但连接失败（`webIp` 仍为 `0.0.0.0`），自动降级回串口模式——这种优雅降级保证用户不会因网络问题而无法操作设备。`GlobalState` 是全局单例，贯穿整个固件保存当前模式、引脚映射、频率等运行时状态。

终端类型确定后，根据类型构建不同的视图/输入并启动分发器：

```cpp
    switch (terminalType) {
        case TerminalTypeEnum::SerialPort: {
            SerialTerminalView serialView(hostSerial);
            SerialTerminalInput serialInput(hostSerial);
            auto baud = std::to_string(state.getSerialTerminalBaudRate());
            serialView.setBaudrate(state.getSerialTerminalBaudRate());
            DependencyProvider* provider = new DependencyProvider(serialView, deviceView, serialInput, deviceInput,
                                                                  littleFsService);
            ActionDispatcher dispatcher(*provider);
            dispatcher.setup(terminalType, baud);
            dispatcher.run();
            break;
        }
        case TerminalTypeEnum::WiFiAp:
        case TerminalTypeEnum::WiFiClient: {
            httpd_handle_t server = nullptr;
            httpd_config_t config = HTTPD_DEFAULT_CONFIG();
            config.lru_purge_enable = true;
            config.recv_wait_timeout = 11;
            config.send_wait_timeout = 11;

            if (terminalType == TerminalTypeEnum::WiFiAp) {
                DnsServer::configureCaptiveDns(config);
                DnsServer::startCaptiveDns(webIp);
            }
            if (httpd_start(&server, &config) != ESP_OK) {
                return;
            }

            JsonTransformer jsonTransformer;
            HttpServer httpServer(server, littleFsService, jsonTransformer);
            WebSocketServer wsServer(server);
            WebTerminalView webView(wsServer);
            WebTerminalInput webInput(wsServer);
            deviceView.loading();
            delay(7000);

            wsServer.setupRoutes();
            httpServer.setupRoutes();
            if (terminalType == TerminalTypeEnum::WiFiAp) {
                httpServer.setupCaptivePortalRoutes(webIp);
            }

            DependencyProvider* provider = new DependencyProvider(webView, deviceView, webInput, deviceInput,
                                                                  littleFsService);
            ActionDispatcher dispatcher(*provider);
            dispatcher.setup(terminalType, webIp);
            dispatcher.run();
            break;
        }
```

这段代码展现了两种终端路径的对称设计。串口路径直接用 `HostSerial` 构建视图和输入；Wi-Fi 路径则启动 ESP-IDF 的 `httpd` 服务器，配置 `lru_purge_enable`（启用 URI 缓存 LRU 淘汰以节省内存），并设置 11 秒的收发超时以适应浏览器交互的延迟。AP 模式下额外启动 DNS 俘获门户——用户连上热点后任何域名都会被重定向到设备 IP，免去手动输入地址的步骤。注意 `DependencyProvider` 被 `new` 到堆上而非栈上，注释说明"too big to fit on the stack anymore"——随着控制器和接口增多，依赖图已超出栈空间，改用堆分配。无论哪种路径，最终都构造 `ActionDispatcher` 并调用 `dispatcher.run()` 进入永久循环，`loop()` 保持空置。

### 2. 中央命令分发器（ActionDispatcher.cpp）

`ActionDispatcher` 是固件的中枢神经，负责读取用户输入、解析命令语法、路由到对应协议控制器。它支持别名展开、字节码指令、宏命令、重复命令和管道命令五种输入形态。

```cpp
void ActionDispatcher::run() {
    while (true) {
        const auto mode = ModeEnumMapper::toString(state.getCurrentMode());
        provider.getTerminalView().printPrompt(mode);

        const std::string action =
            provider.getCommandLineManager().readCommand(mode);

        if (!action.empty()) {
            dispatch(action);
        }
    }
}
```

主循环极为简洁：打印当前模式提示符（如 `HiZ>` 或 `I2C>`），通过 `CommandLineManager` 读取一行完整命令，然后交给 `dispatch()` 处理。提示符中嵌入模式名让用户随时知道当前处于哪条协议总线。

```cpp
void ActionDispatcher::dispatch(const std::string& raw) {
    if (raw.empty()) return;

    const std::string& finalRaw = provider.getAliasManager().expand(raw);

    if (provider.getInstructionTransformer().isInstructionCommand(finalRaw)) {
        auto instructions = provider.getInstructionTransformer().transform(finalRaw);
        dispatchInstructions(instructions);
        return;
    }
    if (provider.getCommandTransformer().isMacroCommand(finalRaw)) {
        provider.getTerminalView().println("Macros Not Yet Implemented.");
        return;
    }
    if (provider.getCommandTransformer().isRepeatCommand(finalRaw)) {
        dispatchRepeatCommands(finalRaw);
        return;
    }
    if (provider.getCommandTransformer().isPipelineCommand(finalRaw)) {
        dispatchPipelineCommands(finalRaw);
        return;
    }

    TerminalCommand cmd = provider.getCommandTransformer().transform(finalRaw);
    dispatchCommand(cmd);
}
```

`dispatch()` 实现了一条优先级管线。首先用 `AliasManager` 展开别名——用户可以为常用命令设置缩写。然后依次判断是否为字节码指令（Bus Pirate 风格的 `[0x55 0xAA]` 格式）、宏命令、重复命令（如 `3: scan` 表示执行 3 次扫描）和管道命令（用 `|` 串联多条命令）。只有都不匹配时，才转换为 `TerminalCommand` 对象走单命令路径。这种分层判断让脚本引擎和交互命令共用同一套底层控制器。

```cpp
void ActionDispatcher::dispatchCommand(const TerminalCommand& cmd) {
    if (cmd.getRoot() == "mode" || cmd.getRoot() == "m") {
        ModeEnum maybeNewMode = provider.getUtilityController().handleModeChangeCommand(cmd);
        if (maybeNewMode != ModeEnum::None) {
            setCurrentMode(maybeNewMode);
        }
        return;
    }
    if (provider.getCommandTransformer().isGlobalCommand(cmd)) {
        provider.getUtilityController().handleCommand(cmd);
        if (provider.getCommandTransformer().isScreenCommand(cmd)) {
            setCurrentMode(state.getCurrentMode());
        }
        return;
    }

    switch (state.getCurrentMode()) {
        case ModeEnum::OneWire:
            provider.getOneWireController().handleCommand(cmd); break;
        case ModeEnum::UART:
            provider.getUartController().handleCommand(cmd); break;
        case ModeEnum::I2C:
            provider.getI2cController().handleCommand(cmd); break;
        case ModeEnum::SPI:
            provider.getSpiController().handleCommand(cmd); break;
        case ModeEnum::SUBGHZ:
            provider.getSubGhzController().handleCommand(cmd); break;
        case ModeEnum::RFID:
            provider.getRfidController().handleCommand(cmd); break;
        case ModeEnum::LORA:
            provider.getLoRaController().handleCommand(cmd); break;
        // ... 其余模式 ...
        case ModeEnum::HIZ:
            provider.getTerminalView().println("Type 'help' or 'mode'"); break;
    }
}
```

单命令分发先处理两个全局情况：`mode`/`m` 命令切换协议模式（HiZ→I2C→SPI...），全局命令（`help`、`logic` 等）交给 `UtilityController`。之后用一个巨大的 `switch` 按当前模式路由到对应控制器。这套设计的关键在于 `DependencyProvider`——它持有所有 23 个控制器的引用，`switch` 中每条分支通过 `provider.getXxxController()` 取出对应控制器并调用 `handleCommand(cmd)`。控制器内部再根据 `cmd.getRoot()`（命令根词）分发到具体操作函数。屏幕相关命令执行后还会调用 `setCurrentMode` 重新渲染引脚映射视图。

### 3. WebSocket 传输层（WebSocketServer.cpp）

Web CLI 模式下，浏览器与设备之间通过 WebSocket 双向通信。`WebSocketServer` 封装了帧收发、字符缓冲和 UTF-8 净化。

```cpp
void WebSocketServer::setupRoutes() {
    static httpd_uri_t ws_uri = {
        .uri = "/ws",
        .method = HTTP_GET,
        .handler = WebSocketServer::wsHandler,
        .user_ctx = this,
        .is_websocket = true
    };
    httpd_register_uri_handler(server, &ws_uri);
}
```

路由注册将 `/ws` 路径绑定到 WebSocket 处理器。`is_websocket = true` 告诉 ESP-IDF 的 HTTP 服务器对该路径执行 WebSocket 协议升级握手。

```cpp
esp_err_t WebSocketServer::wsHandler(httpd_req_t *req) {
    WebSocketServer* self = static_cast<WebSocketServer*>(req->user_ctx);

    if (req->method == HTTP_GET) {
        int newClientFd = httpd_req_to_sockfd(req);
        if (clientFd >= 0 && clientFd != newClientFd) {
            closeClient(self->server, clientFd);
        }
        clientFd = newClientFd;
        return ESP_OK;
    }

    httpd_ws_frame_t frame = {};
    frame.type = HTTPD_WS_TYPE_TEXT;
    frame.payload = nullptr;

    esp_err_t ret = httpd_ws_recv_frame(req, &frame, 0);
    if (ret != ESP_OK) return ret;

    frame.payload = (uint8_t*)malloc(frame.len + 1);
    if (!frame.payload) return ESP_ERR_NO_MEM;

    ret = httpd_ws_recv_frame(req, &frame, frame.len);
    if (ret != ESP_OK) {
        free(frame.payload);
        if (clientFd == httpd_req_to_sockfd(req)) {
            clientFd = -1;
        }
        return ret;
    }
    frame.payload[frame.len] = '\0';

    for (size_t i = 0; i < frame.len; ++i) {
        self->buffer.push_back(((char*)frame.payload)[i]);
    }
    free(frame.payload);
    return ESP_OK;
}
```

这段帧处理采用**两阶段读取**模式：第一次 `httpd_ws_recv_frame` 传入长度 0，仅获取帧头中的 `frame.len`（负载长度），据此 `malloc` 精确大小的缓冲区后第二次读取实际数据。`+1` 是为字符串终止符预留。读取成功后逐字节推入 `buffer`（一个 `std::deque`），将 WebSocket 帧转换为字节流——这样上层 `CommandLineManager` 可以像读串口一样逐字符消费。单客户端设计也很明确：新连接到来时若已有旧客户端，先关闭旧的再记录新的 `clientFd`，保证同一时刻只有一个活跃会话。

```cpp
char WebSocketServer::readCharBlocking() {
    while (buffer.empty()) {
        delay(10);
    }
    char c = buffer.front();
    buffer.pop_front();
    return c;
}
```

阻塞读字符在缓冲区空时以 10ms 间隔轮询，避免忙等。这种设计让 `WebTerminalInput` 的接口与 `SerialTerminalInput` 完全一致——上层无法区分输入来自串口还是浏览器。

```cpp
std::string WebSocketServer::sanitizeUtf8(const std::string& input) {
    std::string output;
    size_t i = 0;
    while (i < input.size()) {
        unsigned char c = input[i];
        if (c <= 0x7F) {
            output += c; i++;
        } else if ((c & 0xE0) == 0xC0 && i + 1 < input.size() &&
                   (input[i+1] & 0xC0) == 0x80) {
            output += input.substr(i, 2); i += 2;
        } else if ((c & 0xF0) == 0xE0 && i + 2 < input.size() &&
                   (input[i+1] & 0xC0) == 0x80 &&
                   (input[i+2] & 0xC0) == 0x80) {
            output += input.substr(i, 3); i += 3;
        } else if ((c & 0xF8) == 0xF0 && i + 3 < input.size() &&
                   (input[i+1] & 0xC0) == 0x80 &&
                   (input[i+2] & 0xC0) == 0x80 &&
                   (input[i+3] & 0xC0) == 0x80) {
            output += input.substr(i, 4); i += 4;
        } else {
            i++;
        }
    }
    return output;
}
```

`sanitizeUtf8` 在发送前逐字节验证 UTF-8 编码合法性。它按照 RFC 3629 规则检查前导字节的高位模式（`0xxxxxxx`、`110xxxxx`、`1110xxxx`、`11110xxx`）及后续字节是否以 `10xxxxxx` 开头。非法字节被直接跳过——这防止了因终端转义序列或损坏数据导致浏览器端渲染异常。对于嵌入式设备而言，这种防御性编码保证了串口终端中可能混入的原始二进制数据不会破坏 Web 界面。

### 4. 逻辑分析仪（PinAnalyzer.cpp）

`PinAnalyzer` 将 ESP32-S3 变成一台基础逻辑分析仪，通过轮询采样引脚电平，检测边沿、统计脉冲宽度、识别突发信号和提取 PWM/伺服周期。

```cpp
void PinAnalyzer::begin(uint8_t pin_) {
    end();
    pulseRing = new uint32_t[PULSE_RING];
    riseUs    = new uint32_t[RISE_RING];

    pin = pin_;
    pinService.setInput(pin);

    lastLevel = pinService.read(pin);
    startLevel = lastLevel;

    windowStartMs = utilityService.nowMs();
    lastChangeUs = utilityService.nowUs();

    edges = 0;
    highUs = 0;
    lowUs  = 0;
    minPulseUs = 0xFFFFFFFF;
    maxPulseUs = 0;
    basePulseUs = 0;

    bursts = 0;
    burstEdges = 0;
    maxGapUs = 0;
    maxGapWasHigh = false;
    inBurst = false;

    pulseCount = 0;
    pulseHead = 0;
    riseCount = 0;
    riseHead = 0;
    lastRiseUs = 0;
    lastHighPulseUs = 0;
}
```

`begin` 初始化两个环形缓冲区：`pulseRing` 存储每个脉冲的持续时间（微秒），`riseUs` 存储上升沿时间戳用于计算信号周期。初始化时记录起始电平、窗口起始时间和上次边沿时间，所有统计量归零。`minPulseUs` 初始化为 `0xFFFFFFFF`（uint32 最大值）确保第一个采样脉冲必然小于它。

```cpp
void PinAnalyzer::sample() {
    bool v = pinService.read(pin);
    if (v == lastLevel) return;
    onEdge(v, utilityService.nowUs());
}

void PinAnalyzer::onEdge(bool newLevel, uint32_t nowUs) {
    uint32_t dt = nowUs - lastChangeUs;

    if (lastLevel) highUs += dt;
    else           lowUs  += dt;

    if (dt < minPulseUs) minPulseUs = dt;
    if (dt > maxPulseUs) maxPulseUs = dt;

    pulseRing[pulseHead] = dt;
    pulseHead = (pulseHead + 1) % PULSE_RING;
    if (pulseCount < PULSE_RING) pulseCount++;

    const uint32_t gapThresholdUs = 5000;
    if (dt >= gapThresholdUs) {
        if (inBurst) inBurst = false;
        bursts++;
        if (dt > maxGapUs) {
            maxGapUs = dt;
            maxGapWasHigh = lastLevel;
        }
    } else {
        inBurst = true;
        burstEdges++;
    }

    if (!lastLevel && newLevel) {
        riseUs[riseHead] = nowUs;
        riseHead = (riseHead + 1) % RISE_RING;
        if (riseCount < RISE_RING) riseCount++;
        lastRiseUs = nowUs;
    }
    if (lastLevel && !newLevel) {
        lastHighPulseUs = dt;
    }

    edges++;
    lastLevel = newLevel;
    lastChangeUs = nowUs;
}
```

`sample` 采用轮询采样——每次调用读取引脚电平，与 `lastLevel` 不同则触发 `onEdge`。这种轮询方式虽然不及硬件中断精确，但在 ESP32-S3 的高主频下对于 kHz 级信号足够实用。`onEdge` 是分析核心：`dt` 计算本次脉冲持续时间，据此累计高电平和低电平总时间。脉冲宽度写入环形缓冲区 `pulseRing`，用取模实现循环覆盖，保留最近的脉冲序列。

突发检测以 5ms（5000μs）为间隙阈值：脉冲间隔超过 5ms 视为信号组之间的间隙，`bursts` 计数加一；否则视为同一突发组内的连续边沿。这能有效区分红外遥控的引导脉冲组与数据帧组、或 SubGHz 信号的数据包。

上升沿时间戳存入 `riseUs` 环形缓冲区，后续 `collectRisePeriods` 通过相邻上升沿之差计算信号周期——这正是 PWM/伺服控制信号的基波周期。下降沿时记录 `lastHighPulseUs`，即高电平持续时间，用于占空比计算。

```cpp
void PinAnalyzer::collectPulses(std::vector<uint32_t>& out) const {
    out.clear();
    out.reserve(pulseCount);
    int start = (pulseHead - pulseCount);
    if (start < 0) start += PULSE_RING;
    for (int i = 0; i < (int)pulseCount; i++) {
        int idx = (start + i) % PULSE_RING;
        out.push_back(pulseRing[idx]);
    }
}
```

`collectPulses` 从环形缓冲区中按时间顺序（旧→新）提取脉冲序列。由于写入时 `pulseHead` 不断前进，起始位置需要回溯 `pulseCount` 个位置并处理负值回绕。输出的脉冲序列可用于协议解码——例如红外 NEC 协议的 562.5μs 载波周期、或 WS2812 LED 的 0 码 350ns/1 码 700ns 时序。

### 5. I2C 协议控制器（I2cController.cpp）

每个协议控制器遵循统一的 `handleCommand` 模式：根据命令根词分发到具体操作函数。I2C 控制器支持扫描、嗅探、读写、转储、故障注入、洪泛、干扰等操作。

```cpp
void I2cController::handleCommand(const TerminalCommand& cmd) {
    if (cmd.getRoot() == "scan") handleScan();
    else if (cmd.getRoot() == "discovery") handleDiscover();
    else if (cmd.getRoot() == "sniff") handleSniff();
    else if (cmd.getRoot() == "ping") handlePing(cmd);
    else if (cmd.getRoot() == "identify") handleIdentify(cmd);
    else if (cmd.getRoot() == "write") handleWrite(cmd);
    else if (cmd.getRoot() == "read") handleRead(cmd);
    else if (cmd.getRoot() == "dump") handleDump(cmd);
    else if (cmd.getRoot() == "regs") handleRegs(cmd);
    else if (cmd.getRoot() == "slave") handleSlave(cmd);
    else if (cmd.getRoot() == "glitch") handleGlitch(cmd);
    else if (cmd.getRoot() == "flood") handleFlood(cmd);
    else if (cmd.getRoot() == "jam") handleJam();
    else if (cmd.getRoot() == "eeprom") handleEeprom(cmd);
    else if (cmd.getRoot() == "recover") handleRecover();
    else if (cmd.getRoot() == "monitor") handleMonitor(cmd);
    else if (cmd.getRoot() == "trace") handleTrace(cmd);
    else if (cmd.getRoot() == "swap") handleSwap();
    else if (cmd.getRoot() == "health") handleHealth(cmd);
    else if (cmd.getRoot() == "config") handleConfig();
    else handleHelp();
}
```

这种 `if-else if` 链按命令根词匹配操作函数。除常规的 `scan`/`read`/`write`/`dump` 外，还包含安全研究向的功能：`glitch`（电压/时序故障注入）、`flood`（地址洪泛）、`jam`（总线干扰）、`slave`（从机模拟）。`eeprom` 会进入 I2C EEPROM 专属 Shell，提供更细粒度的存储操作子命令。未匹配任何命令时调用 `handleHelp` 显示帮助。

```cpp
void I2cController::handleScan() {
    terminalView.println("I2C Scan: Scanning I2C bus... Press [ENTER] to stop");
    terminalView.println("");
    bool found = false;

    for (uint8_t addr = 1; addr < 127; ++addr) {
        char key = terminalInput.readChar();
        if (key == '\r' || key == '\n') {
            terminalView.println("I2C Scan: Cancelled by user.");
            return;
        }

        i2cService.beginTransmission(addr);
        if (i2cService.endTransmission() == 0) {
            std::stringstream ss;
            ss << "Found device at 0x" << std::hex << std::uppercase << (int)addr;
            terminalView.println(ss.str());
            found = true;
        }
    }

    if (!found) {
        terminalView.println("I2C Scan: No I2C devices found.");
    }
}
```

`handleScan` 遍历 7 位 I2C 地址空间（1-126），对每个地址执行 `beginTransmission` + `endTransmission`：返回 0 表示收到 ACK，即该地址有设备响应。每次循环前检查用户是否按下回车，实现可中断扫描。结果以十六进制大写格式输出，如 `Found device at 0x68`。这种实现是 I2C 总线探测的标准手法，与 `i2cdetect` 等工具原理一致。

```cpp
void I2cController::handleSniff() {
    terminalView.println("I2C Sniffer: Listening on SCL/SDA... Press [ENTER] to stop.\n");
    i2c_sniffer_begin(state.getI2cSclPin(), state.getI2cSdaPin());
    if (!i2c_sniffer_setup()) {
        terminalView.println("I2C Sniffer: Not enough memory to allocate buffers.");
        return;
    }

    std::string line;
    while (true) {
        char key = terminalInput.readChar();
        if (key == '\r' || key == '\n') break;

        while (i2c_sniffer_available()) {
            char c = i2c_sniffer_read();
            if (c == '\n') {
                line += "  ";
                terminalView.println(line);
                line.clear();
            } else {
                line += c;
            }
        }
        utilityService.sleepUs(100);
    }

    i2c_sniffer_reset_buffer();
    i2c_sniffer_stop();
    i2cService.configure(state.getI2cSdaPin(), state.getI2cSclPin(), state.getI2cFrequency());
    terminalView.println("\n\nI2C Sniffer: Stopped.");
}
```

嗅探模式通过底层 `i2c_sniffer_*` 系列函数利用 GPIO 边沿中断捕获 SDA/SCL 线上的事务。主循环同时监听两个输入源：用户的按键（回车退出）和嗅探缓冲区的数据。每个字节以十六进制追加到 `line`，遇到换行符（嗅探器标记一次事务结束）时输出整行。100μs 的休眠避免 CPU 满载。退出时先重置缓冲区再停止嗅探，最后用当前引脚和频率配置重新初始化 I2C 服务，为后续正常操作恢复状态。

### 6. SubGHz 射频扫描器（SubGhzController.cpp）

SubGHz 模式操作 CC1101 射频芯片，支持频段扫描、信号重放、暴力破解和信号解码。`handleScan` 实现了完整的频谱扫描流程。

```cpp
void SubGhzController::handleScan(const TerminalCommand& cmd) {
    const auto& args = cmd.getArgs();
    auto bands = subGhzService.getSupportedBand();

    auto bandIndex = userInputManager.readValidatedChoiceIndex("Select frequency band:", bands, 0);
    subGhzService.setScanBand(bands[bandIndex]);
    std::vector<float> freqs = subGhzService.getSupportedFreq(bands[bandIndex]);

    int holdMs = userInputManager.readValidatedInt("Hold time per frequency (ms):", 4, 1, 5000);
    int rssiThr = userInputManager.readValidatedInt("RSSI threshold detection (dBm):", -67, -127, 0);

    if (!subGhzService.applyScanProfile(4.8f, 200.0f, 2 /* OOK */, true)) {
        terminalView.println("SUBGHZ: Not detected. Run 'config' first.");
        return;
    }
```

扫描配置分三步交互：选择频段（如 433MHz、868MHz、915MHz）、设置每个频率的停留时间（默认 4ms，范围 1-5000ms）和 RSSI 检测阈值（默认 -67 dBm）。`applyScanProfile` 以 4.8kBaud 符号率、200kHz 接收带宽、OOK 调制模式初始化 CC1101——这是 SubGHz 遥控器的典型参数。如果 CC1101 未检测到，提示用户先运行 `config` 配置引脚。

```cpp
    std::vector<int>  best(freqs.size(), -127);
    std::vector<bool> wasAbove(freqs.size(), false);
    bool stopRequested = false;

    while (!stopRequested) {
        for (size_t i = 0; i < freqs.size(); ++i) {
            int c = terminalInput.readChar();
            if (c == '\n' || c == '\r') { stopRequested = true; break; }

            float f = freqs[i];
            subGhzService.tune(f);

            int peak = subGhzService.measurePeakRssi(holdMs);
            if (peak > best[i]) best[i] = peak;

            if (peak >= rssiThr && !wasAbove[i]) {
                terminalView.println(" [PEAK] f=" + argTransformer.toFixed2(f) + " MHz  RSSI=" + std::to_string(peak) + " dBm");
                wasAbove[i] = true;
            } else if (peak < rssiThr - 2) {
                wasAbove[i] = false;
            }
        }
    }
```

扫描主循环遍历频段内所有频率，对每个频率调谐 CC1101 并在 `holdMs` 时间内测量峰值 RSSI。`best` 数组记录每个频率的历史最大 RSSI。当信号超过阈值且之前未触发时，实时输出峰值通知（带频率和 dBm 值）；当信号回落到阈值以下 2dB 时重置标志，实现边沿触发避免重复告警。用户随时按回车可中断扫描。

```cpp
    std::vector<size_t> idx(freqs.size());
    std::iota(idx.begin(), idx.end(), 0);
    std::sort(idx.begin(), idx.end(), [&](size_t a, size_t b){ return best[a] > best[b]; });
    terminalView.println("\n [SCAN] Best peaks:");
    const size_t n = std::min<size_t>(5, idx.size());
    for (size_t k = 0; k < n; ++k) {
        size_t i = idx[k];
        terminalView.println("   " + argTransformer.toFixed2(freqs[i]) + " MHz  RSSI=" + std::to_string(best[i]) + " dBm");
    }

    if (!idx.empty() && best[idx[0]] > -120) {
        auto confirm = userInputManager.readYesNo(" Save tuning to best frequency ("
            + argTransformer.toFixed2(freqs[idx[0]]) + " MHz)?", true);
        if (!confirm) return;
        subGhzService.tune(freqs[idx[0]]);
        state.setSubGhzFrequency(freqs[idx[0]]);
    }
```

扫描结束后，用 `std::iota` 生成索引序列并按 RSSI 降序排序，输出最强的 5 个频率。如果最强信号高于 -120 dBm（说明确实有有效发射），询问用户是否保存该频率为默认调谐频率。`readYesNo` 的第二个参数 `true` 表示默认选择"是"，方便快速操作。保存后写入 `GlobalState`，后续 `replay`/`receive` 等命令直接使用该频率。

---

## API 使用指南

### Python 串口自动化

项目支持通过 Python 脚本经串口自动化操作，使用 Bus Pirate 兼容的命令语法：

```python
import serial
import time

# 连接设备（根据实际端口修改）
ser = serial.Serial('/dev/ttyACM0', 115200, timeout=1)
time.sleep(2)  # 等待设备就绪

def send(cmd):
    ser.write((cmd + '\n').encode())
    time.sleep(0.3)
    return ser.read_all().decode(errors='replace')

# 切换到 I2C 模式并扫描总线
print(send('mode'))          # 进入模式选择
print(send('i2c'))           # 选择 I2C
print(send('scan'))          # 扫描所有 I2C 设备地址

# 读取 0x68 设备（如 DS1307 RTC）寄存器 0x00
print(send('read 0x68 0x00 1'))   # 读 1 字节

# 切换到 SubGHz 模式扫描频率
print(send('mode'))
print(send('subghz'))
print(send('scan'))

ser.close()
```

脚本通过 `pyserial` 发送文本命令并读取响应，延时 300ms 等待命令执行。`read 0x68 0x00 1` 表示对地址 0x68 的设备从寄存器 0x00 读取 1 字节。所有命令与交互式 CLI 完全一致，实现"所见即所脚本"。

### cURL 访问 Web API

Wi-Fi 模式下设备启动 HTTP 服务器，提供 REST 端点。以下示例假设设备 IP 为 `192.168.4.1`（AP 模式）：

```bash
# 获取系统信息
curl -s http://192.168.4.1/api/sysinfo

# 上传文件到 LittleFS（用于导入数据/脚本）
curl -X POST http://192.168.4.1/api/upload \
  -F "file=@payload.bin"

# 从 LittleFS 下载转储的 EEPROM 数据
curl -s http://192.168.4.1/api/download/dump.hex -o dump.hex
```

Web CLI 本身通过 WebSocket 端点 `/ws` 交互，也可用 `websocat` 等工具直接连接：

```bash
# 通过 WebSocket 发送命令（需要安装 websocat）
echo "mode" | websocat ws://192.168.4.1/ws
```

---

## 编译与部署

### 环境搭建

| 依赖 | 版本要求 | 说明 |
|------|----------|------|
| PlatformIO Core | 最新版 | 构建系统 |
| Python | 3.8+ | PlatformIO 依赖 |
| ESP32-S3 板 | 8MB+ Flash | 任意 ESP32-S3 开发板 |
| USB 数据线 | — | 连接设备 |

### 编译配置（platformio.ini 摘要）

项目为每款板卡定义独立的构建环境，通过 `-D` 宏注入引脚映射。以下是主要环境：

| 环境 (`[env:xxx]`) | 目标板 | Flash/PSRAM | 特色库 |
|------|------|------|------|
| `t-display-s3` | LILYGO T-Display S3 | 16M/8M | LovyanGFX |
| `waveshare-s3-geek` | Waveshare ESP32-S3-GEEK | 16M/2M | LovyanGFX |
| `t-embed-s3` | LILYGO T-Embed | — | LovyanGFX + RotaryEncoder |
| `t-embed-s3-cc1101` | T-Embed CC1101 | — | + CC1101 板载 |
| `xiao-esp32s3` | Seeed XIAO S3 | — | 无屏幕 |
| `vision-master-t190` | Heltec Vision Master T190 | 16M/8M | + 板载 SX1262 |
| `heltec_wifi_lora_32_V4` | Heltec WiFi LoRa 32 V4 | 16M/2M | + 板载 SX1262 |
| `native-tests` | 本机单元测试 | — | Unity 测试框架 |

每个环境的 `build_flags` 包含 `PROTECTED_PINS`（不可用引脚列表，避免与 Flash/USB/屏幕冲突）和每条协议的引脚定义。换板只需选择对应环境编译，引脚映射自动生效。

### 编译与烧录步骤

```bash
# 1. 克隆仓库
git clone -b pioarduino https://github.com/geo-tp/ESP32-Bit-Pirate.git
cd ESP32-Bit-Pirate

# 2. 编译指定板卡固件（以 T-Display S3 为例）
pio run -e t-display-s3

# 3. 烧录到设备
pio run -e t-display-s3 -t upload

# 4. 打开串口监视器
pio device monitor

# 5. 单元测试（本机运行，无需硬件）
pio test -e native-tests
```

### 浏览器 Web 烧录

无需安装任何工具，访问 [Web Flasher](https://geo-tp.github.io/ESP32-Bit-Pirate/webflasher/) 即可在 Chrome/Edge 浏览器中通过 WebSerial 直接刷写固件。M5Stack 设备也可通过 M5Burner 烧录。

### 启动后操作流程

1. 上电后通过屏幕/串口选择终端模式：Serial / WiFi AP / WiFi Client / Standalone
2. Wi-Fi 模式下选择连接方式，AP 模式默认 SSID 可在屏幕上查看
3. 浏览器访问显示的 IP 地址，或用串口终端连接
4. 输入 `mode` 切换协议模式，`help` 查看当前模式命令

---

## 项目亮点与适用场景

### 架构亮点

**接口驱动的依赖注入**：40+ 纯虚接口（`II2cService`、`ISubGhzService` 等）将控制器与硬件访问完全解耦。控制器只依赖接口，具体实现由 `DependencyProvider` 注入。这使得每个协议可以独立测试——`native-tests` 环境用 Mock 实现替换真实硬件，在 PC 上运行 Unity 单元测试。

**终端无关性**：串口视图、Web 视图和 Cardputer 屏幕视图实现同一套 `ITerminalView`/`IInput` 接口，命令分发逻辑完全共享。用户在浏览器中输入 `scan` 和在串口终端输入 `scan`，经过的代码路径完全一致。

**字节码脚本引擎**：支持 Bus Pirate 风格的字节码指令（如 `[ 0x55 0xAA (r) ]` 表示发送两字节后读取一个字节），让自动化测试脚本可以紧凑地描述复杂协议序列。

**多板引脚抽象**：所有协议引脚通过 PlatformIO 编译宏注入，源码中用 `state.getI2cSdaPin()` 等运行时读取，换板零代码改动。

### 适用场景

- **嵌入式开发调试**：I2C/SPI 设备探测、EEPROM 转储、UART 桥接、逻辑分析
- **硬件安全研究**：协议嗅探、信号重放分析、JTAG/SWD 调试、故障注入测试
- **教学演示**：用一块开发板演示 20+ 种总线协议的交互方式
- **物联网原型**：SubGHz/LoRa/RF24/蓝牙射频原型验证、Wi-Fi/以太网网络分析
- **现场维护**：便携式 Cardputer 独立模式实现无电脑现场操作

---

## 总结

ESP32 Bit Pirate 把 Bus Pirate 的理念推向了新高度——不再局限于单一串口终端，而是同时支持 USB 串口、Wi-Fi 浏览器和板载独立三种交互方式，覆盖了从桌面调试到现场便携的全场景。4360 颗 star 和当日活跃的更新节奏印证了社区对这类"一块板搞定所有协议"工具的强烈需求。从代码层面看，接口驱动的分层架构、40+ 抽象接口和 PlatformIO 多板配置展现了成熟的工程实践，使得新增一个协议模式只需实现对应的 Controller、Service、Interface 和 Shell 四件套，扩展性清晰可循。对于任何需要与多种数字总线或射频协议打交道的嵌入式开发者而言，这都是一个值得收藏和深入学习的项目。

---

📝 作者：蔡浩宇（jun-chy）
📅 日期：2026-07-23
🔗 项目地址：https://github.com/geo-tp/ESP32-Bit-Pirate
