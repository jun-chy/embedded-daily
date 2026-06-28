# Sesame Robot — 基于 ESP32 的开源四足机器人深度解析

> 📅 2026-06-28 | 分类：ESP32 | ⭐ 3,075 Stars
> 
> 作者：蔡浩宇（jun-chy）

## 项目链接

- **GitHub**: [dorianborian/sesame-robot](https://github.com/dorianborian/sesame-robot)
- **作者**: Dorian Todd
- **许可证**: Apache-2.0
- **最近更新**: 2026-05-31
- **主要语言**: C (88%)、C++ (8.5%)、Python (3.5%)

## 一、项目简介

Sesame Robot 是一个基于 ESP32 微控制器的开源四足机器人项目，由 Dorian Todd 创建。它使用 8 个 MG90 舵机（每条腿 2 个关节）实现约 8 个自由度的运动，配备 128x64 OLED 屏幕作为表情显示系统，支持 WiFi 远程控制和 RESTful API。整个机器人可通过 3D 打印制作，硬件成本约 $50-60，仅需基础焊接技能和 Arduino IDE 即可完成组装。

### 核心特性一览

| 特性 | 说明 |
|------|------|
| 四足步态 | 8 个 MG90 舵机，每条腿 2 个关节（髋 + 膝） |
| 表情系统 | 128x64 SSD1306 OLED，运动同步 + 对话表情 + 空闲动画 |
| WiFi 双模式 | AP（热点直连）+ STA（接入家庭网络）同时运行 |
| RESTful API | JSON 接口，支持 Python/JS/cURL 程序化控制 |
| Captive Portal | DNS 劫持，连上 WiFi 自动弹出控制页面 |
| mDNS | 通过 `sesame-robot.local` 访问，无需记忆 IP |
| 动画编辑器 | Sesame Studio 桌面应用，可视化设计动作序列 |
| 模拟器 | Rust 编写的 3D 物理仿真环境 |
| 语音助手集成 | 对话表情 + talk 变体，配合 Python 实现语音控制 |

## 二、硬件架构

### 2.1 系统组成

```
┌─────────────────────────────────────────────────┐
│                 ESP32 (主控)                     │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐ │
│  │ WiFi     │  │ I2C      │  │ PWM (4 Timer) │ │
│  │ AP+STA   │  │ SDA/SCL  │  │ 8ch × 50Hz    │ │
│  └────┬─────┘  └────┬─────┘  └──────┬────────┘ │
│       │             │               │           │
└───────┼─────────────┼───────────────┼───────────┘
        │             │               │
   ┌────┴────┐  ┌─────┴─────┐  ┌─────┴──────────┐
   │ WiFi AP │  │ SSD1306   │  │ 8x MG90 Servo  │
   │ 192.168 │  │ 128x64    │  │ R1 R2 L1 L2     │
   │ .4.1    │  │ OLED      │  │ R4 R3 L3 L4     │
   └─────────┘  └───────────┘  └────────────────┘
```

### 2.2 舵机命名与布局

固件定义了一套清晰的舵机命名体系，将 8 个舵机按左右和功能编号：

```cpp
enum ServoName : uint8_t {
    R1 = 0,   // 右腿1 - 髋关节
    R2 = 1,   // 右腿1 - 膝关节
    L1 = 2,   // 左腿1 - 髋关节
    L2 = 3,   // 左腿1 - 膝关节
    R4 = 4,   // 右腿2 - 膝关节
    R3 = 5,   // 右腿2 - 髋关节
    L3 = 6,   // 左腿2 - 髋关节
    L4 = 7    // 左腿2 - 膝关节
};

const String ServoNames[] = {"R1","R2","L1","L2","R4","R3","L3","L4"};
```

**解析**: 命名中的数字代表功能层级——`1/2` 为前腿对，`3/4` 为后腿对。每个关节通过 `servoNameToIndex()` 函数支持字符串名称到数组索引的映射，方便通过 Serial CLI 调试单个舵机。

### 2.3 多板适配的引脚配置

固件通过条件注释支持四种不同的 ESP32 开发板，用户只需取消注释对应配置：

```cpp
// Sesame Distro Board V3 (ESP32-S3) Pinout [NEW]
//const int servoPins[8] = {4, 5, 6, 7, 10, 11, 12, 13};

// Sesame Distro Board V2 (ESP32-S3) Pinout (Legacy)
//const int servoPins[8] = {4, 5, 6, 7, 15, 16, 17, 18};

// Sesame Distro Board V1 (ESP32-WROOM32) Pinout (Legacy)
//const int servoPins[8] = {15, 2, 23, 19, 4, 16, 17, 18};

// Lolin S2 Mini (ESP32-S2) Pinout
const int servoPins[8] = {1, 2, 4, 6, 8, 10, 13, 14};

// I2C Pins (OLED 显示屏)
#define I2C_SDA 33   // S2 Mini
#define I2C_SCL 35
//#define I2C_SDA 8    // Distro Board V2/V3
//#define I2C_SCL 9
//#define I2C_SDA 21   // Distro Board V1
//#define I2C_SCL 22
```

**解析**: 不同板子的引脚分配差异源于 ESP32 变体的 GPIO 可用性不同（如 S2 Mini 的引脚较少，S3 有更多 PWM 通道）。移植到其他 ESP32 变体时，只需确保选择的引脚支持 PWM 输出且非输入专用引脚。

### 2.4 硬件清单

| 组件 | 规格 | 数量 | 参考价 |
|------|------|------|--------|
| 主控板 | Lolin S2 Mini / Distro Board V3 | 1 | ~$5 |
| 舵机 | MG90 金属齿轮 180° | 8 | ~$2/个 |
| 显示屏 | SSD1306 128x64 I2C OLED | 1 | ~$3 |
| 电源 | 5V 3A USB-C PD | 1 | ~$5 |
| 3D 打印件 | PLA 材质 | 全身 | ~$10 |
| 紧固件 | 螺丝、螺母、轴承 | 若干 | ~$5 |
| **合计** | | | **~$50-60** |

## 三、固件架构

### 3.1 文件结构

固件采用模块化设计，将逻辑与资源分离：

| 文件 | 大小 | 职责 |
|------|------|------|
| `sesame-firmware-main.ino` | 28KB | 主入口：setup/loop、网络配置、API 端点、系统逻辑 |
| `movement-sequences.h` | 14KB | 所有运动序列和姿态定义 |
| `face-bitmaps.h` | 297KB | OLED 表情位图数据（存储在 Flash） |
| `captive-portal.h` | 27KB | Web 控制界面的 HTML/CSS/JS |

### 3.2 表情动画系统

固件使用 X-Macros 技术实现表情的自动注册，避免手动维护多个 switch 语句：

```cpp
// 每个表情自动生成帧数组
#define MAKE_FACE_FRAMES(name) \
    const unsigned char* const face_##name##_frames[] = { \
        epd_bitmap_##name, epd_bitmap_##name##_1, epd_bitmap_##name##_2, \
        epd_bitmap_##name##_3, epd_bitmap_##name##_4, epd_bitmap_##name##_5 \
    };

// X-Macro 展开 - 自动为 FACE_LIST 中每个表情创建帧数组
#define X(name) MAKE_FACE_FRAMES(name)
FACE_LIST
#undef X

// 自动生成表情查找表
const FaceEntry faceEntries[] = {
#define X(name) { #name, face_##name##_frames, MAX_FACE_FRAMES },
    FACE_LIST
#undef X
    { "default", face_defualt_frames, MAX_FACE_FRAMES }
};
```

**解析**: `FACE_LIST` 宏是一个 X-Macro，在 `face-bitmaps.h` 中定义。添加新表情时只需在 `FACE_LIST` 中加一行 `X(my_new_face)`，编译器会自动生成帧数组声明和查找表条目。这极大降低了添加表情的代码量。

三种动画模式控制表情播放行为：

```cpp
enum FaceAnimMode : uint8_t {
    FACE_ANIM_LOOP = 0,       // 循环播放：到最后一帧后回到第一帧
    FACE_ANIM_ONCE = 1,       // 单次播放：到最后一帧后停止
    FACE_ANIM_BOOMERANG = 2  // 往返播放：正向播放后反向播放
};
```

表情帧率可在运行时动态配置：

```cpp
const FaceFpsEntry faceFpsEntries[] = {
    { "walk", 1 },      // 行走表情：1 FPS
    { "point", 5 },     // 指向表情：5 FPS
    { "idle_blink", 7 },// 眨眼动画：7 FPS
    { "dead", 2 },      // 死亡表情：2 FPS
    // 对话表情（手动控制，不自动动画）
    { "happy", 1 },
    { "talk_happy", 1 },
    { "sad", 1 },
    { "talk_sad", 1 },
    // ... 更多
};
```

## 四、核心代码深度分析

### 4.1 PWM 舵机控制与防 Brownout 设计

ESP32 的硬件 PWM 定时器资源有限，固件需要精心分配：

```cpp
void setup() {
    // 分配全部 4 个硬件定时器给 PWM 使用
    ESP32PWM::allocateTimer(0);
    ESP32PWM::allocateTimer(1);
    ESP32PWM::allocateTimer(2);
    ESP32PWM::allocateTimer(3);
    
    for (int i = 0; i < 8; i++) {
        servos[i].setPeriodHertz(50);  // 50Hz = 20ms 周期，标准舵机频率
        // 将 0-180 度映射到 732-2929 微秒脉冲宽度
        servos[i].attach(servoPins[i], 732, 2929);
    }
}
```

**解析**: ESP32 有 4 个通用硬件定时器，每个可驱动多个 PWM 通道。50Hz 对应 20ms 的周期，标准舵机控制脉冲宽度范围为 500-2500us。这里使用 732-2929us 是为了最大化 MG90 的 180 度行程。如果要使用 270 度舵机，需改为 833-2167us。

防 Brownout 的交错激活机制是固件的关键设计：

```cpp
int motorCurrentDelay = 20;  // 舵机间延迟，默认 20ms

void setServoAngle(uint8_t channel, int angle) {
    // 写入角度 + 微调偏移
    servos[channel].write(angle + servoSubtrim[channel]);
    // 关键：在下一个舵机启动前等待，避免瞬时电流过大
    delay(motorCurrentDelay);
}
```

**解析**: 8 个舵机同时启动时瞬时电流可达数安培，可能导致 ESP32 电压骤降而复位。`motorCurrentDelay` 在每个 `setServoAngle` 调用后引入延迟，使舵机依次激活而非同时。如果电源功率充足（如独立 5V 3A 电源），可将此值设为 0 以加快运动速度。该参数可在运行时通过 Web API 动态调整。

### 4.2 双模式 WiFi 与 Captive Portal

固件的核心网络功能是同时运行 AP 和 STA 模式：

```cpp
// --- 网络配置 ---
#define AP_SSID  "Sesame-Controller"
#define AP_PASS  "12345678"
#define NETWORK_SSID ""        // 留空则不连接外部网络
#define NETWORK_PASS ""
#define ENABLE_NETWORK_MODE false

void setup() {
    if (ENABLE_NETWORK_MODE && String(NETWORK_SSID).length() > 0) {
        // 双模式：同时作为热点和客户端
        WiFi.mode(WIFI_AP_STA);
        WiFi.setHostname(deviceHostname.c_str());
        WiFi.begin(NETWORK_SSID, NETWORK_PASS);
        
        // 最多等待 10 秒（20 × 500ms）
        int attempts = 0;
        while (WiFi.status() != WL_CONNECTED && attempts < 20) {
            delay(500);
            Serial.print(".");
            attempts++;
        }
        
        if (WiFi.status() == WL_CONNECTED) {
            networkConnected = true;
            networkIP = WiFi.localIP();
            Serial.print("Connected to network! IP: ");
            Serial.println(networkIP);
        } else {
            // 连接失败，回退到仅 AP 模式
            WiFi.mode(WIFI_AP);
            Serial.println("Failed to connect. Running in AP-only mode.");
        }
    } else {
        WiFi.mode(WIFI_AP);
    }
    
    // 创建接入点
    WiFi.softAP(AP_SSID, AP_PASS);
    IPAddress myIP = WiFi.softAPIP();
    Serial.print("AP Created. IP: ");
    Serial.println(myIP);
}
```

**解析**: `WIFI_AP_STA` 模式让 ESP32 同时作为 AP（供手机直连控制）和 STA（接入家庭 WiFi 供远程访问）。如果 STA 连接失败，自动回退到 AP-only 模式确保基本功能可用。10 秒超时避免启动时卡死。

Captive Portal 的 DNS 劫持实现：

```cpp
DNSServer dnsServer;
const byte DNS_PORT = 53;

void setup() {
    // ...
    // 通配符 "*" 将所有域名解析重定向到 ESP32 的 IP
    dnsServer.start(DNS_PORT, "*", myIP);
    
    // mDNS 服务发现
    if (MDNS.begin(deviceHostname.c_str())) {
        MDNS.addService("http", "tcp", 80);
    }
    
    // 所有未注册路径都返回控制页面
    server.onNotFound(handleRoot);
}
```

**解析**: DNS Server 的通配符模式将所有 DNS 查询（如 `google.com`、`example.com`）都解析为 ESP32 的网关 IP（192.168.4.1）。当手机连上 WiFi 后打开任意网址，浏览器会自动跳转到控制页面——这就是 captive portal 的工作原理。`server.onNotFound(handleRoot)` 确保任何 URL 都返回控制界面。

### 4.3 JSON API 与智能指令解析

固件提供两套 API：传统的 URL 参数接口和现代的 JSON 接口：

```cpp
// 传统接口（URL 参数）
GET /cmd?go=forward       // 向前走
GET /cmd?go=backward      // 向后走
GET /cmd?pose=wave        // 挥手
GET /cmd?stop             // 停止
GET /cmd?motor=1&value=90 // 单舵机控制

// JSON API（推荐）
GET  /api/status          // 获取状态
POST /api/command         // 发送指令
```

JSON API 的核心是智能区分"仅表情更新"和"表情+运动"指令：

```cpp
void handleApiCommand() {
    if (server.method() != HTTP_POST) {
        server.send(405, "application/json", "{\"error\":\"Method not allowed\"}");
        return;
    }
    
    String body = server.arg("plain");
    Serial.println("API Command received:");
    Serial.println(body);
    
    // 检测是否包含 face 字段
    int faceOnlyStart = body.indexOf("\"face\":\"");
    if (faceOnlyStart == -1) {
        faceOnlyStart = body.indexOf("\"face\": \"");
    }
    
    // 如果有 face 但没有 command，则是仅表情更新
    bool faceOnly = (faceOnlyStart > 0 && 
                     body.indexOf("\"command\":") == -1);
    
    String command = "";
    String face = "";
    
    // 解析 face 字段
    if (faceOnlyStart > 0) {
        faceOnlyStart = body.indexOf("\"", faceOnlyStart + 6) + 1;
        int faceEnd = body.indexOf("\"", faceOnlyStart);
        if (faceEnd > faceOnlyStart) {
            face = body.substring(faceOnlyStart, faceEnd);
        }
    }
    
    // 解析 command 字段（如果不是仅表情模式）
    if (!faceOnly) {
        int cmdStart = body.indexOf("\"command\":\"");
        if (cmdStart == -1) {
            cmdStart = body.indexOf("\"command\": \"");
        }
        if (cmdStart == -1) {
            server.send(400, "application/json", "{\"error\":\"Missing command field\"}");
            return;
        }
        cmdStart = body.indexOf("\"", cmdStart + 10) + 1;
        int cmdEnd = body.indexOf("\"", cmdStart);
        command = body.substring(cmdStart, cmdEnd);
    }
    
    // 设置表情
    if (face.length() > 0) {
        setFace(face);
    }
    
    // 仅表情更新 - 不触发运动
    if (faceOnly) {
        recordInput();
        server.send(200, "application/json", 
            "{\"status\":\"ok\",\"message\":\"Face updated\"}");
        return;
    }
    
    // 执行运动指令
    if (command == "stop") {
        currentCommand = "";
    } else {
        currentCommand = command;
        exitIdle();  // 退出空闲动画
    }
    server.send(200, "application/json", 
        "{\"status\":\"ok\",\"message\":\"Command executed\"}");
}
```

**解析**: 这段手动 JSON 解析避免了引入 ArduinoJSON 库的依赖。`faceOnly` 的判断逻辑是：如果请求中有 `face` 字段但没有 `command` 字段，则仅更新 OLED 表情而不触发舵机运动。这在语音助手场景下极为重要——对话时只需要改变表情（如 happy、thinking），而不需要机器人移动。

状态查询接口返回完整运行状态：

```cpp
void handleGetStatus() {
    String json = "{";
    json += "\"currentCommand\":\"" + currentCommand + "\",";
    json += "\"currentFace\":\"" + currentFaceName + "\",";
    json += "\"networkConnected\":" + String(networkConnected ? "true" : "false") + ",";
    json += "\"apIP\":\"" + WiFi.softAPIP().toString() + "\"";
    if (networkConnected) {
        json += ",\"networkIP\":\"" + networkIP.toString() + "\"";
    }
    json += "}";
    server.send(200, "application/json", json);
}
```

### 4.4 非阻塞运动控制

传统的 `delay()` 会阻塞整个主循环，导致 HTTP 请求超时和 OLED 动画卡顿。固件使用自定义函数解决这一问题：

```cpp
// 在等待期间持续处理网络请求和动画更新
bool pressingCheck(String cmd, int ms) {
    unsigned long start = millis();
    while (millis() - start < ms) {
        dnsServer.processNextRequest();  // 处理 DNS 请求
        server.handleClient();            // 处理 HTTP 请求
        updateAnimatedFace();             // 更新表情动画
        updateWifiInfoScroll();           // 更新 WiFi 信息滚动
        
        // 检测指令是否被改变（如用户按了 stop）
        if (currentCommand != cmd) return false;
    }
    return true;
}

// 类似的带表情更新的延迟函数
void delayWithFace(unsigned long ms) {
    unsigned long start = millis();
    while (millis() - start < ms) {
        dnsServer.processNextRequest();
        server.handleClient();
        updateAnimatedFace();
    }
}
```

**解析**: `pressingCheck` 在运动动画的每一帧之间调用。它不仅替代了 `delay()`，还实现了运动可中断性——如果用户在机器人行走时按下"停止"，`currentCommand` 会被清空，`pressingCheck` 检测到 `cmd != currentCommand` 后立即返回 false，运动函数收到 false 后停止执行剩余帧。

### 4.5 步态运动序列分析

以行走步态为例，分析四足机器人的运动学实现：

```cpp
inline void runWalkPose() {
    Serial.println(F("WALK FWD"));
    setFaceWithMode("walk", FACE_ANIM_ONCE);
    
    // 初始姿势：抬起对角腿对
    setServoAngle(R3, 135); setServoAngle(L3, 45);
    setServoAngle(R2, 100); setServoAngle(L1, 25);
    if (!pressingCheck("forward", frameDelay)) return;
    
    for (int i = 0; i < walkCycles; i++) {
        // 帧1：右后腿向前迈步
        setServoAngle(R3, 135); setServoAngle(L3, 0);
        if (!pressingCheck("forward", frameDelay)) return;
        
        // 帧2：左前腿和右前腿调整
        setServoAngle(L4, 135); setServoAngle(L2, 90);
        setServoAngle(R4, 0); setServoAngle(R1, 180);
        if (!pressingCheck("forward", frameDelay)) return;
        
        // 帧3：右前髋关节调整
        setServoAngle(R2, 45); setServoAngle(L1, 90);
        if (!pressingCheck("forward", frameDelay)) return;
        
        // 帧4：脚落地
        setServoAngle(R4, 45); setServoAngle(L4, 180);
        if (!pressingCheck("forward", frameDelay)) return;
        
        // 帧5：另一侧腿抬起
        setServoAngle(R3, 180); setServoAngle(L3, 45);
        setServoAngle(R2, 90); setServoAngle(L1, 0);
        if (!pressingCheck("forward", frameDelay)) return;
        
        // 帧6：完成一个步态周期
        setServoAngle(L2, 135); setServoAngle(R1, 90);
        if (!pressingCheck("forward", frameDelay)) return;
    }
    
    runStandPose(1);  // 回到站立姿势
}
```

**解析**: 步态采用对角步态（Trot Gait）——对角线的两条腿同时运动（R3+L3 为一组，R1+L2 为另一组）。每个步态周期包含 6 个关键帧，通过 `walkCycles` 控制重复次数。每个帧之间调用 `pressingCheck` 确保运动可被实时中断。

挥手动作展示了复合动画序列：

```cpp
inline void runWavePose() {
    Serial.println(F("WAVE"));
    setFaceWithMode("wave", FACE_ANIM_ONCE);
    
    runStandPose(0);  // 先站立（不显示表情）
    delayWithFace(200);
    
    // 抬起右前腿
    setServoAngle(R4, 80); setServoAngle(L3, 180);
    setServoAngle(L2, 90); setServoAngle(R1, 100);
    delayWithFace(200);
    setServoAngle(L3, 180);
    delayWithFace(300);
    
    // 挥手动作：来回摆动 4 次
    for (int i = 0; i < 4; i++) {
        setServoAngle(L3, 180); delayWithFace(300);  // 高位
        setServoAngle(L3, 100); delayWithFace(300);  // 低位
    }
    
    runStandPose(1);  // 回到站立
    if (currentCommand == "wave") currentCommand = "";
}
```

### 4.6 空闲动画系统

机器人无人操作时显示拟人化的空闲行为：

```cpp
// 空闲状态变量
bool idleActive = false;
bool idleBlinkActive = false;
unsigned long nextIdleBlinkMs = 0;
uint8_t idleBlinkRepeatsLeft = 0;

// 进入空闲模式
void enterIdle() {
    idleActive = true;
    // 使用 BOOMERANG 模式播放"呼吸"动画
    setFaceWithMode("idle", FACE_ANIM_BOOMERANG);
    scheduleNextIdleBlink(3000, 7000);  // 3-7 秒后首次眨眼
}

// 退出空闲模式
void exitIdle() {
    idleActive = false;
    idleBlinkActive = false;
}

// 更新空闲眨眼
void updateIdleBlink() {
    if (!idleActive || idleBlinkActive) return;
    
    if (millis() >= nextIdleBlinkMs) {
        idleBlinkActive = true;
        setFace("idle_blink");
        
        // 30% 概率连续眨两次
        idleBlinkRepeatsLeft = (random(100) < 30) ? 1 : 0;
    }
}
```

**解析**: 空闲系统通过随机化眨眼间隔（3-7 秒）和 30% 概率的双眨眼，模拟真实生物的不规则行为，使机器人看起来更"有生命感"。BOOMERANG 动画模式让"呼吸"表情在正向和反向之间来回播放，产生平缓的呼吸效果。

### 4.7 主循环架构

```cpp
void loop() {
    // === 网络处理 ===
    dnsServer.processNextRequest();  // Captive Portal DNS
    server.handleClient();           // HTTP 请求处理
    
    // === 显示更新 ===
    updateAnimatedFace();            // 表情动画帧推进
    updateIdleBlink();               // 空闲眨眼检查
    updateWifiInfoScroll();          // WiFi 信息滚动（前30秒）
    
    // === 运动执行 ===
    if (currentCommand != "") {
        String cmd = currentCommand;
        if (cmd == "forward")      runWalkPose();
        else if (cmd == "backward") runWalkBackward();
        else if (cmd == "left")     runTurnLeft();
        else if (cmd == "right")    runTurnRight();
        else if (cmd == "rest")     { runRestPose(); currentCommand = ""; }
        else if (cmd == "stand")    { runStandPose(1); currentCommand = ""; }
        else if (cmd == "wave")     runWavePose();
        else if (cmd == "dance")    runDancePose();
        else if (cmd == "swim")     runSwimPose();
        else if (cmd == "point")    runPointPose();
        else if (cmd == "pushup")   runPushupPose();
        else if (cmd == "bow")      runBowPose();
        else if (cmd == "cute")     runCutePose();
        else if (cmd == "freaky")   runFreakyPose();
        else if (cmd == "worm")     runWormPose();
        else if (cmd == "shake")    runShakePose();
        else if (cmd == "shrug")    runShrugPose();
        else if (cmd == "dead")     runDeadPose();
        else if (cmd == "crab")     runCrabPose();
    }
    
    // === 串口调试 CLI ===
    if (Serial.available()) {
        // 支持单舵机调试命令
    }
}
```

**解析**: 主循环采用单核事件循环架构，按优先级依次处理网络、显示和运动。持续型指令（forward/backward/left/right）会在每个 loop 迭代中重新执行，直到 `currentCommand` 被清空。一次性指令（wave/dance 等）执行完毕后自动清空 `currentCommand`。

## 五、API 使用指南

### 5.1 Python 控制

```python
import requests
import time

robot_url = "http://sesame-robot.local"

# 获取状态
status = requests.get(f"{robot_url}/api/status").json()
print(f"当前表情: {status['currentFace']}")
print(f"网络连接: {'是' if status['networkConnected'] else '否'}")

# 让机器人挥手 + 开心表情
requests.post(f"{robot_url}/api/command", json={
    "command": "wave",
    "face": "happy"
})

time.sleep(3)

# 仅更新表情（不运动）
requests.post(f"{robot_url}/api/command", json={"face": "excited"})

# 停止运动
requests.post(f"{robot_url}/api/command", json={"command": "stop"})
```

### 5.2 语音助手集成

```python
import speech_recognition as sr
from textblob import TextBlob

def analyze_sentiment(text):
    """分析语义情感，返回对应表情"""
    polarity = TextBlob(text).sentiment.polarity
    if polarity > 0.5:   return "excited"
    elif polarity > 0.2: return "happy"
    elif polarity < -0.5: return "angry"
    elif polarity < -0.2: return "sad"
    else:                return "thinking"

def set_robot_face(face, talking=False):
    """更新表情，talking=True 时使用 talk_ 变体"""
    if talking:
        face = f"talk_{face}"
    requests.post(f"{robot_url}/api/command", json={"face": face})

recognizer = sr.Recognizer()
while True:
    with sr.Microphone() as source:
        set_robot_face("idle")
        audio = recognizer.listen(source, timeout=5)
        
        set_robot_face("thinking")
        text = recognizer.recognize_google(audio)
        
        emotion = analyze_sentiment(text)
        set_robot_face(emotion, talking=True)  # 说话表情
        time.sleep(2)
        set_robot_face(emotion, talking=False)  # 静默表情
```

### 5.3 cURL 快速测试

```bash
# 获取状态
curl http://sesame-robot.local/api/status

# 让机器人跳舞
curl -X POST http://sesame-robot.local/api/command \
  -H "Content-Type: application/json" \
  -d '{"command":"dance","face":"dance"}'

# 仅切换表情
curl -X POST http://sesame-robot.local/api/command \
  -H "Content-Type: application/json" \
  -d '{"face":"surprised"}'

# 调整参数
curl "http://sesame-robot.local/setSettings?frameDelay=120&motorCurrentDelay=25"
```

## 六、编译与部署

### 6.1 环境搭建

1. **安装 Arduino IDE 2.0+**
2. **添加 ESP32 板支持**:
   - File → Preferences → Additional Board Manager URLs
   - 添加: `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`
   - Boards Manager → 搜索 "ESP32" → 安装 "ESP32 by Espressif Systems" v2.0.0+
3. **安装依赖库**（Library Manager）:
   - `ESP32Servo` **v3.0.9**（重要：更新版本有已知多舵机干扰 bug [#103](https://github.com/madhephaestus/ESP32Servo/issues/103)）
   - `Adafruit SSD1306`
   - `Adafruit GFX Library`

### 6.2 编译配置

| 开发板 | Board 选项 | 关键设置 |
|--------|-----------|---------|
| Lolin S2 Mini | LOLIN S2 Mini | Upload Speed: 921600, USB CDC On Boot: Enabled |
| Distro Board V3 | ESP32S3 Dev Module | USB CDC On Boot: Enabled, Flash Mode: QIO 80MHz |
| Distro Board V1 | ESP32 Dev Module | 默认配置 |

Partition Scheme 统一选择: `Default 4MB with spiffs`

### 6.3 烧录步骤

1. USB 连接开发板
2. 打开 `sesame-firmware-main.ino`（确保同目录包含所有 .h 文件）
3. 根据板子取消注释对应的 `servoPins` 和 I2C 引脚配置
4. （可选）配置网络模式：修改 `NETWORK_SSID` 和 `NETWORK_PASS`
5. 选择 COM 端口
6. 点击 Upload

### 6.4 使用方法

**AP 模式（默认）**:
- 连接 WiFi: `Sesame-Controller-BETA`，密码: `12345678`
- 打开浏览器访问任意网址，自动跳转到控制界面

**网络模式**:
- 修改代码中的 WiFi 配置并重新烧录
- 通过 `http://sesame-robot.local` 或串口显示的 IP 访问

**串口调试**:
- 打开 Serial Monitor（115200 baud）
- 支持命令: `rn wf`（行走）、`motor 1 90`（单舵机控制）等

## 七、项目亮点与适用场景

### 技术亮点

1. **防 Brownout 交错激活**: 8 个舵机依次启动，避免电源过载导致复位
2. **非阻塞运动控制**: `pressingCheck` 在动画帧间处理网络请求，运动可实时中断
3. **X-Macro 表情系统**: 添加表情只需一行代码，自动生成帧数组和查找表
4. **双模式 WiFi**: AP + STA 同时运行，直连和网络远程控制两不误
5. **Captive Portal**: DNS 劫持实现零配置访问体验
6. **Face-Only API**: 智能区分表情更新和运动指令，适配语音助手场景
7. **拟人化空闲**: 随机眨眼 + 双眨眼 + 呼吸动画

### 适用场景

- **教育学习**: ESP32 PWM/I2C/WiFi/运动学综合实践
- **语音助手**: 对话表情 + 动作响应的物理化身
- **IoT 集成**: JSON API 作为可编程的物理交互节点
- **创客入门**: 低成本四足机器人，社区活跃
- **二次开发**: 添加传感器（超声波、陀螺仪）扩展功能

## 八、总结

Sesame Robot 是一个完成度极高的开源四足机器人项目，从硬件 CAD、固件、Web UI、动画编辑器到模拟器形成了完整生态。其 ESP32 固件在多个技术点上有值得学习的设计：PWM 定时器分配与交错激活防 brownout、双模式 WiFi + captive portal 的网络架构、X-Macro 表情自动注册系统、以及 `pressingCheck` 非阻塞可中断运动控制。对于想要入门机器人开发的嵌入式工程师，这是一个兼顾趣味性和技术深度的优秀起点项目。

---

> 📝 作者：蔡浩宇（jun-chy）
> 
> 📅 日期：2026-06-28
> 
> 🔗 项目地址：https://github.com/dorianborian/sesame-robot
