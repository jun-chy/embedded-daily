# Sesame Robot — 基于 ESP32 的开源四足机器人

> 📅 2026-06-28 | 分类：ESP32 | ⭐ 3,075 Stars

## 项目链接

- **GitHub**: [dorianborian/sesame-robot](https://github.com/dorianborian/sesame-robot)
- **作者**: Dorian Todd
- **许可证**: Apache-2.0
- **最近更新**: 2026-05-31

## 项目简介

Sesame Robot 是一个基于 ESP32 微控制器的开源四足机器人项目。它使用 8 个舵机（每条腿 2 个）实现约 8 个自由度的运动，配备 128x64 OLED 屏幕作为表情显示，支持 WiFi 远程控制和 RESTful API。整个机器人可通过 3D 打印制作，硬件成本约 $50-60，适合各种技能水平的创客和工程师。

### 核心特性

- **四足设计**: 8 个 MG90 舵机驱动，每条腿 2 个关节
- **表情显示**: 128x64 OLED 屏幕，与运动同步的动画表情系统
- **全 3D 打印**: 设计为 PLA 材质打印，最少量支撑
- **网络连接**: WiFi 双模式（AP + STA），支持远程控制和 API
- **JSON API**: RESTful 接口，支持 Python/JavaScript 等编程控制
- **动画编辑器**: Sesame Studio 桌面应用，可视化设计动作
- **空闲动画系统**: 随机眨眼、呼吸动画等拟人化行为

## 技术栈分析

| 维度 | 详情 |
|------|------|
| 主控芯片 | ESP32-S2 (Lolin S2 Mini) / ESP32-S3 (Distro Board V3) |
| 开发框架 | Arduino-ESP32 |
| 舵机驱动库 | ESP32Servo v3.0.9 |
| 显示库 | Adafruit SSD1306 + Adafruit GFX |
| 网络协议 | WiFi (AP+STA)、mDNS、HTTP RESTful API |
| DNS 服务 | DNSServer（ captive portal） |
| 开发环境 | Arduino IDE 2.0+ |
| 主要语言 | C (88%)、C++ (8.5%)、Python (3.5%) |

## 核心代码分析

### 1. 硬件初始化与引脚配置

固件通过 `servoPins` 数组抽象引脚定义，支持多种开发板配置：

```cpp
Servo servos[8];

// Lolin S2 Mini Pinout
const int servoPins[8] = {1, 2, 4, 6, 8, 10, 13, 14};

// Sesame Distro Board V3 Pinout (ESP32-S3)
//const int servoPins[8] = {4, 5, 6, 7, 10, 11, 12, 13};

// Sesame Distro Board V1 Pinout (ESP32-WROOM32)
//const int servoPins[8] = {15, 2, 23, 19, 4, 16, 17, 18};

// I2C Pins for S2 Mini Board
#define I2C_SDA 33
#define I2C_SCL 35
```

**解析**: 使用条件编译注释实现多板适配，用户只需取消注释对应板子的配置即可。I2C 引脚用于 OLED 显示屏通信。

### 2. PWM 舵机控制与交错激活

为了防止多个舵机同时启动导致电压崩溃（brownout），固件采用交错激活策略：

```cpp
// PWM Init
ESP32PWM::allocateTimer(0);
ESP32PWM::allocateTimer(1);
ESP32PWM::allocateTimer(2);
ESP32PWM::allocateTimer(3);

for (int i = 0; i < 8; i++) {
    servos[i].setPeriodHertz(50);
    // Map 0-180 to approx 732-2929us
    servos[i].attach(servoPins[i], 732, 2929);
}
```

```cpp
void setServoAngle(uint8_t channel, int angle) {
    // motorCurrentDelay 防止同时驱动多个舵机导致电流过载
    servos[channel].write(angle + servoSubtrim[channel]);
    delay(motorCurrentDelay); // 默认 20ms
}
```

**解析**: `allocateTimer()` 分配 4 个硬件定时器，50Hz PWM 频率对应舵机标准控制频率。脉冲宽度映射为 732us-2929us，覆盖大多数 180 度 hobby 舵机。`motorCurrentDelay` 在每个舵机写入间引入延迟，避免 8 个舵机同时运动时瞬时电流过大导致 ESP32 复位。

### 3. 双模式 WiFi 与 Captive Portal

固件同时运行 AP（接入点）和 STA（站点）模式，并通过 DNS 劫持实现 captive portal：

```cpp
// 双模式 WiFi
if (ENABLE_NETWORK_MODE && String(NETWORK_SSID).length() > 0) {
    WiFi.mode(WIFI_AP_STA); // 同时启用 AP 和 STA
    WiFi.setHostname(deviceHostname.c_str());
    WiFi.begin(NETWORK_SSID, NETWORK_PASS);
    
    int attempts = 0;
    while (WiFi.status() != WL_CONNECTED && attempts < 20) {
        delay(500);
        attempts++;
    }
    
    if (WiFi.status() == WL_CONNECTED) {
        networkConnected = true;
        networkIP = WiFi.localIP();
    } else {
        WiFi.mode(WIFI_AP); // 回退到 AP-only
    }
} else {
    WiFi.mode(WIFI_AP);
}

// 创建 AP
WiFi.softAP(AP_SSID, AP_PASS);

// mDNS 服务发现
if (MDNS.begin(deviceHostname.c_str())) {
    MDNS.addService("http", "tcp", 80);
}

// DNS 劫持 - 将所有域名解析重定向到 ESP32 IP
dnsServer.start(DNS_PORT, "*", myIP);
```

**解析**: `WIFI_AP_STA` 模式让机器人同时作为热点（供直连控制）和客户端（接入家庭网络）。DNSServer 的通配符 `"*"` 将所有 DNS 查询重定向到 ESP32 的网关 IP（192.168.4.1），实现 captive portal——用户连接 WiFi 后打开任意网址都会自动跳转到控制界面。mDNS 允许通过 `sesame-robot.local` 访问，无需记忆 IP。

### 4. JSON API 与指令处理

固件提供 RESTful JSON API，支持外部程序化控制：

```cpp
void handleApiCommand() {
    if (server.method() != HTTP_POST) {
        server.send(405, "application/json", "{\"error\":\"Method not allowed\"}");
        return;
    }
    
    String body = server.arg("plain");
    
    // 检测 face-only 命令（无 command 字段，仅更新表情）
    int faceOnlyStart = body.indexOf("\"face\":\"");
    bool faceOnly = (faceOnlyStart > 0 && 
                     body.indexOf("\"command\":") == -1);
    
    // 解析 face 字段
    String face = "";
    if (faceOnlyStart > 0) {
        int faceEnd = body.indexOf("\"", faceOnlyStart + 7);
        face = body.substring(faceOnlyStart + 8, faceEnd);
    }
    
    // 设置表情
    if (face.length() > 0) {
        setFace(face);
    }
    
    // 仅表情更新
    if (faceOnly) {
        server.send(200, "application/json", 
            "{\"status\":\"ok\",\"message\":\"Face updated\"}");
        return;
    }
    
    // 执行运动指令
    if (command == "stop") {
        currentCommand = "";
    } else {
        currentCommand = command;
        exitIdle();
    }
    server.send(200, "application/json", 
        "{\"status\":\"ok\",\"message\":\"Command executed\"}");
}
```

**解析**: API 智能区分"仅表情更新"和"表情+运动"两种请求。当请求中只有 `face` 字段而无 `command` 字段时，仅更新 OLED 表情而不触发舵机运动——这对语音助手集成特别有用，可以在对话时实时改变表情而不让机器人移动。

### 5. 非阻塞运动控制

使用自定义 `pressingCheck` 函数替代 `delay()`，在动画执行期间持续处理网络请求：

```cpp
bool pressingCheck(String cmd, int ms) {
    unsigned long start = millis();
    while (millis() - start < ms) {
        dnsServer.processNextRequest();
        server.handleClient();
        updateAnimatedFace();
        // 检测指令是否变化（如用户按下 stop）
        if (currentCommand != cmd) return false;
    }
    return true;
}
```

**解析**: 传统 `delay()` 会阻塞整个主循环，导致网络请求超时。`pressingCheck` 在等待期间持续处理 DNS 请求、HTTP 客户端和表情动画更新，使得运动可以被实时中断——例如用户在机器人行走时按下"停止"，能立即响应。

### 6. 主循环与指令分发

```cpp
void loop() {
    dnsServer.processNextRequest();
    server.handleClient();
    updateAnimatedFace();
    updateIdleBlink();
    updateWifiInfoScroll();
    
    if (currentCommand != "") {
        String cmd = currentCommand;
        if (cmd == "forward") runWalkPose();
        else if (cmd == "backward") runWalkBackward();
        else if (cmd == "left") runTurnLeft();
        else if (cmd == "right") runTurnRight();
        else if (cmd == "wave") runWavePose();
        else if (cmd == "dance") runDancePose();
        // ... 更多动作
    }
}
```

**解析**: 主循环采用事件驱动架构，依次处理 DNS、HTTP、动画更新、空闲检测，然后执行当前运动指令。`currentCommand` 作为状态变量，非空时持续执行对应动作序列，设为空时停止运动。

## 编译与部署方法

### 前置条件

1. 安装 Arduino IDE 2.0+
2. 添加 ESP32 板支持包 URL:
   ```
   https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json
   ```
3. 安装依赖库:
   - `ESP32Servo` **v3.0.9**（注意：不要用更新版本，有已知多舵机干扰 bug）
   - `Adafruit SSD1306`
   - `Adafruit GFX Library`

### 编译步骤

1. 打开 `sesame-firmware-main.ino`（需与同名文件夹一起）
2. 选择开发板:
   - Lolin S2 Mini → "LOLIN S2 Mini"
   - Distro Board V3 → "ESP32S3 Dev Module"
3. 配置:
   - Upload Speed: 921600
   - USB CDC On Boot: Enabled
   - Partition Scheme: Default 4MB with spiffs
4. 选择正确的 COM 端口
5. 根据板子取消注释对应的 `servoPins` 和 I2C 引脚
6. 点击 Upload 按钮

### 使用方法

- **AP 模式**: 连接 WiFi `Sesame-Controller-BETA`（密码 `12345678`），浏览器打开任意网址进入控制界面
- **网络模式**: 修改代码中的 `NETWORK_SSID` 和 `NETWORK_PASS`，通过 `http://sesame-robot.local` 访问
- **API 调用示例**:
  ```bash
  # 让机器人跳舞
  curl -X POST http://sesame-robot.local/api/command \
    -H "Content-Type: application/json" \
    -d '{"command":"dance","face":"dance"}'
  
  # 仅更新表情
  curl -X POST http://sesame-robot.local/api/command \
    -H "Content-Type: application/json" \
    -d '{"face":"happy"}'
  ```

## 项目亮点与适用场景

### 亮点

1. **低成本可复制**: ~$60 即可制作一个四足机器人，全 3D 打印结构
2. **完善的软件生态**: 固件 + Web UI + 动画编辑器 + 模拟器 + Python 伴侣应用
3. **表情系统丰富**: 包含运动表情、对话表情（含 talk 变体）和空闲动画
4. **API-first 设计**: JSON API 让机器人可作为 IoT 设备集成到更大系统
5. **防 brownout 设计**: 交错激活舵机，避免电源过载

### 适用场景

- **教育**: 学习 ESP32 PWM、I2C、WiFi 和运动学
- **语音助手集成**: 对话表情 + 动作响应，配合 Python 实现语音控制
- **IoT 平台**: 作为可编程的物理交互节点
- **创客项目**: 低成本入门四足机器人

## 总结

Sesame Robot 是一个完成度极高的开源四足机器人项目，从硬件设计、固件、Web UI 到动画工具形成了完整生态。其 ESP32 固件在舵机控制（交错激活防 brownout）、网络通信（双模式 WiFi + captive portal）、API 设计（face-only 与 movement 分离）等方面都有值得学习的设计。对于想要入门机器人开发的嵌入式工程师，这是一个很好的起点项目。

---

*本文档由 embedded-daily 自动生成系统于 2026-06-28 创建*
