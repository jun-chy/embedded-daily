# Bramble：ESP32-S3 隐私优先 LoRa 网状网络固件深度解析

> **日期**：2026-07-18 ｜ **分类**：ESP32 / ESP-IDF / LoRa 网状网络 ｜ **Stars**：1 ｜ **作者**：蔡浩宇（jun-chy）

## 项目链接

| 项目信息 | 内容 |
|---------|------|
| GitHub 地址 | [justinlindh/bramble](https://github.com/justinlindh/bramble) |
| 作者 | justinlindh |
| 许可证 | MIT |
| 最近更新 | 2026-07-18 |
| 主要语言 | C |
| 代码规模 | ~16.9 MB（含硬件设计、模拟器、Web 客户端） |
| 默认分支 | main |

## 项目简介

Bramble 是一套面向 ESP32-S3 + SX1262 的加密、多跳 LoRa 网状网络协议与固件栈，专为无基础设施的长距离通信设计。它聚焦于受限、有损 RF 环境下的实用私密通信，并在协议层面显式处理路由、重传、空口时间（airtime）预算与节点身份。

与 Meshtastic、MeshCore 类系统相比，Bramble 的差异点在于：将隐私和"可认证的、可确认投递"作为协议级一等公民。项目明确声明处于 pre-alpha 阶段，所有特性主张都附带诚实的边界说明（例如模拟器在 50–100 节点时投递率会塌缩到 10–12%）。

### 核心特性一览

| 特性 | 说明 |
|------|------|
| 双 substrate 路由 | 默认反应式 AODV（RREQ/RREP 发现、缓存路由表、反向路由面包屑、中间节点 RREP、路由转发 ACK）；可选泛洪（hop-limited、去重、空口预算门控） |
| 认证流量（wire v4） | DATA 帧与控制面（RREP/RERR/ACK/回执/beacon）携带 network-key HMAC，中继节点在动作前校验；无公共默认密钥，未配钥节点 fail-closed 静默 |
| 确认投递 | 三级可靠性：Broadcast（fire-and-forget）、Normal（3 次重传 + 退避）、Critical（8 次重传 + 回执携带中继路径） |
| 端到端 DM | 每对 peer 使用 AES-256-GCM 会话，密钥由 X25519 quad-DH 交换派生，前向保密由 HKDF 对称棘轮提供，DH 棘轮用于失陷后恢复 |
| 加密通道 | AES-256-GCM (AEAD)，channel ID 藏在密文内部，多跳经泛洪中继 |
| 加密节点身份 | Ed25519 身份，4 字节地址 = `SHA-256(ed25519_pub)[0:4]`，地址冒充是原像搜索而非 TOFU 竞争 |
| 空口预算 | 所有 TX 走单一预算门控路径，真实 time-on-air 计费，按层 token bucket，区域占空比上限（如 EU 868 1%） |
| 离线邮箱 | 离线节点重新入网时收取排队消息 |
| 位置共享 | 三档隐私：presence / zone（~1km） / exact，AES-256-GCM 端到端加密并填充到定长 |
| 浏览器烧录 | 直接从网页烧录固件，无需工具链 |

## 硬件架构

Bramble 运行在 ESP32-S3 + Semtech SX1262 LoRa 收发器上，支持多块开发板与一块自研 PCB（Bramble Pager v1）。

### 系统组成图

```
                       ┌──────────────────────────────────────┐
                       │           ESP32-S3-WROOM-1            │
                       │  ┌────────────┐   ┌──────────────┐    │
                       │  │  Dual-core │   │   8MB Flash   │    │
                       │  │   Xtensa   │   │  (partitions) │    │
                       │  └──────┬─────┘   └──────┬───────┘    │
                       │         │  PSRAM (8MB)  │             │
                       │  ┌──────┴────────────────┴──────────┐ │
                       │  │  ESP-IDF v5.x  +  Bramble 固件     │ │
                       │  │  ┌──────┐ ┌──────┐ ┌───────────┐  │ │
                       │  │  │mesh_ │ │radio │ │  crypto   │  │ │
                       │  │  │task  │ │abstr │ │ (mbedTLS) │  │ │
                       │  │  └──────┘ └──┬───┘ └───────────┘  │ │
                       │  └──────────────┼────────────────────┘ │
                       └─────────────────┼──────────────────────┘
                                         │ SPI + DIOx + RST
                          ┌──────────────┴──────────────┐
                          │     SX1262 LoRa Transceiver  │
                          │   868/915 MHz  ·  SF7..SF12  │
                          │   TCXO / DC-DC / DIO2 RF SW  │
                          └──────────────┬───────────────┘
                                         │ 433/868/915 MHz
                                  [LoRa Mesh Air Interface]
                                         │
                       ┌─────────────────┴─────────────────┐
                       │     远端 Bramble 节点 (N 跳)        │
                       └───────────────────────────────────┘

  外设:  SSD1306 OLED / ST7789 LCD(LVGL) / SSD1680 e-paper
         GT911 触摸 + I2C 键盘 (T-Deck) / 按钮 / 蜂鸣器 + 振动
         ATGM336H GPS / L76K GNSS / 电池 ADC / I2S 音频(T-Deck)
```

### 引脚配置（Heltec WiFi LoRa 32 V3，典型目标）

| 信号 | ESP32-S3 引脚 | 说明 |
|------|--------------|------|
| SX1262 SCK | GPIO5 | SPI 时钟 |
| SX1262 MISO | GPIO3 | SPI 主入从出 |
| SX1262 MOSI | GPIO6 | SPI 主出从入 |
| SX1262 NSS | GPIO7 | SPI 片选 |
| SX1262 RST | GPIO8 | 复位 |
| SX1262 BUSY | GPIO13 | 忙信号 |
| SX1262 DIO1 | GPIO14 | 中断 1 |
| OLED SDA / SCL | GPIO18 / GPIO17 | I2C 显示 |
| 按键 / LED | board 配置 | 用户输入 |

### 硬件清单表

| 板型 | MCU | 显示 | 输入 | 射频 | 状态 |
|------|-----|------|------|------|------|
| Heltec WiFi LoRa 32 V3 | ESP32-S3 | 0.96" SSD1306 OLED | 按钮 | SX1262 | 运行 |
| Heltec WiFi LoRa 32 V4 | ESP32-S3 | OLED + L76K GNSS | 按钮 | SX1262 | 运行 |
| LilyGo T-Deck Plus | ESP32-S3 | ST7789 320×240 (LVGL v9) | GT911 触摸 + I2C 键盘 | SX1262 (TCXO) | 运行（完整 GUI） |
| Bramble Pager v1 | ESP32-S3-WROOM-1 | 2.13" GDEY0213B74 e-paper | 按钮 | SX1262 + DIO2 RF SW | 设计完成 |

## 固件架构

Bramble 以 ESP-IDF component 形式组织，协议逻辑、射频抽象、安全、可靠性、UI/控制层之间边界清晰，使核心协议可在 host 与模拟器上先行验证。

### 关键目录与模块职责

| 路径 | 模块 | 职责 |
|------|------|------|
| `main/main.c` | 应用入口 | `app_main`、事件分发、连接性轮询、堆诊断 |
| `main/mesh_task.c` (334KB) | 网状任务 | AODV 路由、帧收发、邻居表、调度核心 |
| `main/rpc_methods.c` | RPC 方法 | WebSocket JSON-RPC 接口实现 |
| `main/ws_server.c` | WS 服务器 | 浏览器/Web 客户端连接通道 |
| `main/cli.c` | 命令行 | 串口/主机 CLI 命令 |
| `components/radio/sx1262.c` | SX1262 驱动 | 寄存器配置、收发、CAD |
| `components/radio/tx_gate.c` | TX 门控 | 单一预算门控 + LBT + beacon 预留 |
| `components/airtime/airtime_budget.c` | 空口预算 | 4 层 token bucket + 自适应画像 + 占空比上限 |
| `components/reliability/reliability.c` | 可靠性 | 三级重传、pending ACK 表、退避 |
| `components/dedup/dedup.c` | 去重 | packet_id 去重 + LRU 淘汰 |
| `components/network_key/network_key.c` | 网络密钥 | fail-closed 配钥、HMAC MAC |
| `components/identity/identity.c` | 节点身份 | Ed25519 身份、地址派生、TOFU 钉扎 |
| `components/crypto/crypto_esp.c` | 加密 | mbedTLS 封装：AES-256-GCM、HMAC-SHA256、X25519 |
| `components/dm_session/dm_session.c` | DM 会话 | quad-DH、HKDF 棘轮、前向保密 |
| `components/mailbox/mailbox.c` | 邮箱 | 离线消息存储转发 |
| `components/display/ssd1306.c` / `ssd1680_*.c` / `st7789.c` | 显示驱动 | OLED / e-paper / LCD |
| `components/ota/ota.c` | OTA | 固件空中升级 + 回滚 |
| `simulator/` / `emulator/` | 仿真 | 协议级模拟器 + 真实固件二进制仿真 |

## 核心代码深度分析

以下代码均取自仓库 `main` 分支真实源码，逐段解析。

### 1. 应用入口与事件驱动架构（`main/main.c`）

Bramble 的 `main.c` 采用事件驱动模型：底层状态变化通过 JSON-RPC 通知推送给 Web 客户端，同时做节流以避免 RF/GPS 抖动刷屏。

```c
static void emit_gps_event(const char* event, const bramble_position_t* pos) {
    cJSON* params = cJSON_CreateObject();
    if (!params)
        return;
    cJSON_AddStringToObject(params, "event", event);
    if (pos) {
        cJSON_AddBoolToObject(params, "valid", pos->valid);
        cJSON_AddNumberToObject(params, "lat", pos->latitude_e7 / 1e7);
        cJSON_AddNumberToObject(params, "lon", pos->longitude_e7 / 1e7);
        cJSON_AddNumberToObject(params, "alt_m", pos->altitude_m);
        cJSON_AddNumberToObject(params, "accuracy_m", pos->accuracy_m);
    }
    rpc_notify("bramble.onGpsEvent", params);
    cJSON_Delete(params);
}
```

**逐段解析**：函数构造一个 cJSON 对象作为 JSON-RPC 的 `params`，把 GPS 事件名与位置字段塞入。注意 `latitude_e7 / 1e7`——位置在内部以 e7（整数度×1e7）存储以避免浮点累计误差，仅在对外通知时转浮点。`rpc_notify` 是单向通知（无 id 的 JSON-RPC notification），发给所有已连接的 WebSocket 客户端，最后 `cJSON_Delete` 释放对象防止内存泄漏。

```c
static void on_gps_fix(const bramble_position_t* pos, void* ctx) {
    (void)ctx;
    if (pos && pos->valid) {
        s_last_gps_fix = true;
        static uint64_t s_last_gps_notify_us = 0;
        uint64_t now_us = (uint64_t)esp_timer_get_time();
        if (s_last_gps_notify_us == 0 || (now_us - s_last_gps_notify_us) >= 5000000ULL) {
            s_last_gps_notify_us = now_us;
            emit_gps_event("fix_acquired", pos);
            ESP_LOGI(TAG, "GPS position updated: lat=%.6f lon=%.6f alt=%d", pos->latitude_e7 / 1e7,
                     pos->longitude_e7 / 1e7, pos->altitude_m);
        }
    }
}
```

**逐段解析**：注释点明一个真实重构教训——此回调曾把每个 fix 拷进一个无人读取的本地 location_manager，反而饿死了 UI 真正读取的 manager。重构后这里只做事件发射。`static uint64_t s_last_gps_notify_us` 是函数内静态节流变量：GPS 一分钟可能产生 ~60 次 fix，这里用 5 秒（5000000 µs）窗口节流 RPC 通知与日志，避免串口/WebSocket 被 GPS 抖动淹没。`(void)ctx;` 显式忽略未使用参数，符合 MISRA 风格。

```c
conn_mode_t conn_mode_get(void) {
    nvs_handle_t nvs;
    uint8_t mode = CONN_MODE_WIFI;
    if (nvs_open(NVS_NS_BRAMBLE, NVS_READONLY, &nvs) == ESP_OK) {
        nvs_get_u8(nvs, NVS_KEY_CONN_MODE, &mode);
        nvs_close(nvs);
    }
    if (mode == CONN_MODE_BOTH || mode >= CONN_MODE_COUNT)
        mode = CONN_MODE_WIFI;
    return conn_mode_resolve_boot((conn_mode_t)mode, true);
}
```

**逐段解析**：连接模式持久化在 NVS（非易失存储）中，默认 `CONN_MODE_WIFI`。注释解释了一个关键的资源约束——ESP32-S3 上同时跑 WiFi + BLE 会耗尽内部 SRAM，所以默认只开 WiFi。`mode >= CONN_MODE_COUNT` 是防御性枚举边界检查，防止 NVS 中存了非法值。最后 `conn_mode_resolve_boot` 在板级硬件能力基础上做启动期解析。

### 2. 空口预算门控——Bramble 最核心的 TX 路径（`components/radio/tx_gate.c`）

这是 Bramble 最具工程价值的部分：**所有发送都必须穿过单一预算门控路径**，配合真实 time-on-air 计费、LBT（先听后说）与 beacon 预留。

```c
#define TX_GATE_LBT_MAX_ATTEMPTS 3u
#define TX_GATE_LBT_BACKOFF_BASE_MS 50u
#define TX_GATE_LBT_BACKOFF_MAX_MS 300u

#define TX_GATE_FALLBACK_SF 9u
#define TX_GATE_FALLBACK_BW_HZ 125000u
#define TX_GATE_FALLBACK_CR 1u

void tx_gate_init(tx_gate_t* g, const tx_gate_ops_t* ops, uint8_t max_duty_cycle_pct,
                  bool duty_cycle_enforced) {
    g->ops = *ops;
    airtime_budget_init(&g->budget, g->ops.now_ms());
    airtime_budget_set_duty_cap(&g->budget, max_duty_cycle_pct, duty_cycle_enforced);
    g->beacon_wire_len = 0u;
}
```

**逐段解析**：`tx_gate_t` 通过 `ops` 表注入硬件操作（`now_ms`、`channel_busy`、`transmit`、`wdt_feed`、`random_u32` 等），使门控逻辑可在 host/simulator 上用 mock ops 测试。`TX_GATE_FALLBACK_*` 是无线电未配置时（启动窗）的 ToA 计算回退参数，匹配 freq_plan 默认（SF9/125kHz/CR4-5），保证 boot 早期数学有定义。初始化时把占空比上限交给 `airtime_budget`。

**kind → tier 映射**是预算分层的核心：

```c
uint8_t tx_gate_kind_tier(tx_kind_t kind) {
    switch (kind) {
    case TX_KIND_ROUTING:
    case TX_KIND_ACK:
        return AIRTIME_TIER_CRITICAL;
    case TX_KIND_BEACON:
    case TX_KIND_DATA_BROADCAST:
    case TX_KIND_PROBE:
        return AIRTIME_TIER_BROADCAST;
    case TX_KIND_RECEIPT:
        return AIRTIME_TIER_RECEIPT;
    case TX_KIND_DATA:
    case TX_KIND_DATA_RETRY:
    case TX_KIND_FORWARD:
    case TX_KIND_MAILBOX:
    case TX_KIND_PROBE_REPLY:
    default:
        return AIRTIME_TIER_NORMAL;
    }
}
```

**逐段解析**：注释阐明设计意图——路由控制（RREQ/RREP/RERR）与 ACK 走 CRITICAL 层，使路由修复与 ACK 投递不会被数据负载饿死（"一个未发送的 ACK 是网格里最贵的包：对端会以完整数据尺寸重传"）。CRITICAL 还可向 NORMAL 借用，所以控制流量在拥塞下最后才降级。广播/beacon/探测归 BROADCAST 层，回执独占 RECEIPT 层以防回执风暴挤占其他流量。

**ToA 成本计算**：

```c
uint32_t tx_gate_cost_ms(tx_gate_t* g, uint8_t wire_len) {
    uint8_t sf = 0; uint32_t bw_hz = 0; uint8_t cr = 0;
    g->ops.get_toa_params(&sf, &bw_hz, &cr);
    if (sf < 5u || sf > 12u || bw_hz == 0u) {
        sf = TX_GATE_FALLBACK_SF; bw_hz = TX_GATE_FALLBACK_BW_HZ; cr = TX_GATE_FALLBACK_CR;
    }
    if (cr < 1u || cr > 4u) cr = TX_GATE_FALLBACK_CR;
    uint32_t us = bramble_calculate_airtime_us(wire_len, sf, bw_hz, cr);
    return (us + 999u) / 1000u; /* ceil: never undercount airtime */
}
```

**逐段解析**：从无线电层取当前 SF/BW/CR 参数，做范围校验后回退。`bramble_calculate_airtime_us` 按 LoRa 物理层公式（前导码 + 有效载荷符号数 × 符号时长）计算真实空口时间。`(us + 999u) / 1000u` 是向上取整除法——**宁可高估也不低估空口**，因为低估会导致占空比超限违法（如 EU 868 1% 法规）。

**beacon 预留机制**保证广播数据突发不会抽干 beacon 所需令牌：

```c
static uint32_t beacon_reserve_ms(tx_gate_t* g, uint8_t tier, tx_kind_t kind) {
    if (tier != AIRTIME_TIER_BROADCAST || kind == TX_KIND_BEACON || g->beacon_wire_len == 0u)
        return 0u;
    return tx_gate_cost_ms(g, g->beacon_wire_len);
}
```

**逐段解析**：仅对 BROADCAST 层的非 beacon 流量预留一个 beacon 的 ToA。这样广播数据突发最多把广播层抽干到"剩一个 beacon 的量"，下一个 beacon 定时器触发时令牌必然在场。

**核心发送函数** `tx_gate_transmit`：

```c
int tx_gate_transmit(tx_gate_t* g, const uint8_t* buf, uint8_t len, tx_kind_t kind) {
    uint8_t tier = tx_gate_kind_tier(kind);
    uint32_t cost_ms = tx_gate_cost_ms(g, len);

    airtime_budget_refill(&g->budget, g->ops.now_ms());
    if (!airtime_budget_can_transmit(&g->budget, tier,
                                     cost_ms + beacon_reserve_ms(g, tier, kind))) {
        return TX_GATE_ERR_BUDGET;
    }

    /* Listen-Before-Talk: budget approved, now wait for a quiet channel. */
    for (uint8_t attempt = 0; attempt < TX_GATE_LBT_MAX_ATTEMPTS; attempt++) {
        if (g->ops.wdt_feed) g->ops.wdt_feed();
        if (!g->ops.channel_busy()) break;
        uint32_t backoff_ms = TX_GATE_LBT_BACKOFF_BASE_MS * (1u << attempt);
        if (backoff_ms > TX_GATE_LBT_BACKOFF_MAX_MS) backoff_ms = TX_GATE_LBT_BACKOFF_MAX_MS;
        backoff_ms += g->ops.random_u32() % backoff_ms;
        g->ops.delay_ms(backoff_ms);
    }
    /* After TX_GATE_LBT_MAX_ATTEMPTS, transmit anyway to avoid starvation. */

    if (g->ops.wdt_feed) g->ops.wdt_feed();
    int ret = g->ops.transmit(buf, len);
    if (ret != 0) return TX_GATE_ERR_RADIO;

    airtime_budget_debit(&g->budget, tier, cost_ms);
    return TX_GATE_OK;
}
```

**逐段解析（重点）**：这是整个射频发送的唯一通道，分三段——
1. **预算检查**：先 `refill`（按经过时间补充令牌），再用"成本 + beacon 预留"判断能否发送，不能则直接返回 `TX_GATE_ERR_BUDGET`，从源头拒绝超法规/超预算的发送。
2. **LBT 先听后说**：预算通过后，做最多 3 次 CAD（信道活动检测）尝试，退避是指数退避 `50ms * 2^attempt`，封顶 300ms，并叠加随机抖动 `random_u32() % backoff_ms` 防止多节点同步碰撞。每次循环喂看门狗。3 次后仍忙则**强行发送以避免饿死**——这是吞吐与公平的权衡。
3. **扣费**：只有实际发送成功才 `debit` 扣除令牌，保证失败重试不会重复计费导致预算虚耗。

### 3. 四层 Token Bucket 空口预算（`components/airtime/airtime_budget.c`）

`tx_gate` 的预算后端是一个按节点数量自适应的多层令牌桶。

```c
static uint32_t profile_scale_pct(uint8_t peer_count, int idx) {
    if (peer_count <= 8u) {
        if (idx == AIRTIME_IDX_BROADCAST) return 400u;
        if (idx == AIRTIME_IDX_RECEIPT)   return 500u;
        if (idx == AIRTIME_IDX_NORMAL)    return 300u;
        return 100u; /* critical */
    }
    if (peer_count <= 15u) {
        if (idx == AIRTIME_IDX_BROADCAST || idx == AIRTIME_IDX_RECEIPT) return 250u;
        if (idx == AIRTIME_IDX_NORMAL) return 150u;
        return 100u;
    }
    if (peer_count > 40u) {
        if (idx == AIRTIME_IDX_BROADCAST) return 60u;
        if (idx == AIRTIME_IDX_RECEIPT)   return 50u;
        if (idx == AIRTIME_IDX_NORMAL)    return 75u;
        return 100u;
    }
    return 100u; /* baseline */
}
```

**逐段解析**：预算随 mesh 规模自适应——小 mesh（≤8 节点）给广播/回执/普通层放大 3–5 倍，因为此时碰撞是主要失败模式，多给空口让重传能成功；大 mesh（>40）反过来收窄到 50–75%，因为此时空中拥堵是主要约束。CRITICAL 恒为 100%，保证控制面预算不被放大策略影响。

**占空比法规上限**处理得尤为严谨：

```c
/* Regulatory duty-cycle cap (DES-8): scale all tiers proportionally so
 * ANY 1-hour window stays within the cap, not just the steady state.
 * ... the worst-case window (idle hour fills the bucket, then spend the
 * bucket plus a full hour of refill) transmits capacity + refill =
 * 2 * sum(max_ms). Targeting cap/2 makes that worst case exactly the
 * regulatory cap. ETSI EN 300.220 evaluates any observation window,
 * so capping the steady state alone would still allow a 2x burst hour. */
uint32_t window_target = ab->duty_cap_ms / 2u;
if (ab->duty_enforced && window_target > 0u && total > window_target) {
    for (int i = 0; i < AIRTIME_TIER_COUNT; i++) {
        uint32_t capped =
            (uint32_t)(((uint64_t)ab->max_ms[i] * (uint64_t)window_target) / total);
        ab->max_ms[i] = (capped == 0u) ? 1u : capped;
    }
}
```

**逐段解析（重点）**：这是合规设计的关键洞见——ETSI EN 300.220 评估的是"任意观测窗口"而非稳态。令牌桶的突发容量等于每小时补充量，最坏情况是"空闲一小时装满桶，然后花掉桶 + 一整小时补充"= `2 × sum(max_ms)`。所以把目标设为 `cap/2`，使最坏突发窗口恰好等于法规上限。`uint64_t` 中间运算防止 32 位溢出，`capped == 0u ? 1u` 保证任何层不会被缩到 0 而饿死。

**连续补充模型**：

```c
void airtime_budget_refill(airtime_budget_t* ab, uint32_t now_ms) {
    uint32_t elapsed = now_ms - ab->last_refill_ms;
    if (elapsed == 0u) return;
    for (int i = 0; i < AIRTIME_TIER_COUNT; i++) {
        uint64_t numer = (uint64_t)ab->refill_remainder[i]
                       + ((uint64_t)ab->max_ms[i] * (uint64_t)elapsed);
        uint32_t add = (uint32_t)(numer / AIRTIME_REFILL_INTERVAL_MS);
        ab->refill_remainder[i] = (uint32_t)(numer % AIRTIME_REFILL_INTERVAL_MS);
        if (add > 0u) {
            uint64_t next = (uint64_t)ab->tokens_ms[i] + (uint64_t)add;
            ab->tokens_ms[i] = (next >= ab->max_ms[i]) ? ab->max_ms[i] : (uint32_t)next;
        }
    }
    ...
}
```

**逐段解析**：令牌按 `max_ms × elapsed / REFILL_INTERVAL` 连续补充，`refill_remainder` 保留除法余数，使长时间运行下没有令牌漂移。补充后封顶到 `max_ms`（桶满即止）。这种"持续滴入 + 余数累积"模型比"每间隔整补"更精确，避免边界处多补或少补。

**CRITICAL 跨层借用**：

```c
bool airtime_budget_can_transmit(airtime_budget_t* ab, uint8_t tier, uint32_t airtime_ms) {
    int idx = tier_idx(tier);
    if (ab->tokens_ms[idx] >= airtime_ms) return true;
    if (idx == AIRTIME_IDX_CRITICAL) {
        uint32_t deficit = airtime_ms - ab->tokens_ms[AIRTIME_IDX_CRITICAL];
        if (ab->tokens_ms[AIRTIME_IDX_NORMAL] >= deficit && ab->borrow_tokens_ms >= deficit)
            return true;
    }
    return false;
}
```

**逐段解析**：CRITICAL 层令牌不足时可向 NORMAL 借，但双重门控——NORMAL 余额 + 独立的 `borrow_tokens_ms` 借用配额都须足够。这限制中继控制洪泛不会耗尽本地数据通道：借用配额是单独的桶，借爆了就停。

### 4. 三级可靠性与重传退避（`components/reliability/reliability.c`）

```c
uint8_t tier_max_retries(uint8_t tier) {
    switch (tier) {
    case MSG_TIER_BROADCAST: return 0;
    case MSG_TIER_NORMAL:    return 3;
    case MSG_TIER_CRITICAL:  return 8;
    default:                 return 0;
    }
}

uint32_t tier_base_delay_ms(uint8_t tier) {
    switch (tier) {
    case MSG_TIER_NORMAL:   return 2000;
    case MSG_TIER_CRITICAL: return 3000;
    default:                return 0;
    }
}
```

**逐段解析**：广播 0 重传（fire-and-forget），普通 3 次，关键 8 次。基础退避普通 2s、关键 3s——关键消息反而退避更长，是因为关键消息（如密钥交换）价值高，宁可多等也要让前一次发送有充分时间被确认，避免重传风暴叠加。

```c
int pending_ack_add(pending_ack_table_t* table, uint32_t packet_id, uint32_t dest_addr,
                    uint8_t tier, const uint8_t* packet, uint16_t len, uint32_t now_ms) {
    for (int i = 0; i < MAX_PENDING_ACKS; i++) {
        if (!table->entries[i].active) {
            pending_ack_t* e = &table->entries[i];
            e->packet_id = packet_id; e->dest_addr = dest_addr; e->tier = tier;
            e->attempt = 0;
            e->max_attempts = tier_max_retries(tier);
            uint16_t stored = len > sizeof(e->packet_data) ? (uint16_t)sizeof(e->packet_data) : len;
            e->packet_len = stored;
            if (stored > 0 && packet) memcpy(e->packet_data, packet, stored);
            e->next_retry_ms = now_ms + tier_base_delay_ms(tier);
            e->active = true;
            return i;
        }
    }
    return -1;
}
```

**逐段解析**：待确认表用静态数组 + `active` 标志位管理槽位（无动态分配，适合嵌入式）。关键在 `stored` 的 clamp——注释强调这是"防御性不变量"而非"静默截断合法流量"：合法 DATA 帧构造时已保证 ≤ `PENDING_ACK_MAX_FRAME`，clamp 防止 `packet_len` 越界导致重传消费者（mesh_tx、模拟器桥）读越界缓冲区。`next_retry_ms` 设为首退避起点。

```c
static uint32_t jitter_25(uint32_t base) {
    uint32_t quarter = base / 4;
    if (quarter == 0) return base;
    return base - quarter + (uint32_t)(rand() % (2 * quarter + 1));
}

void pending_ack_tick(pending_ack_table_t* table, uint32_t now_ms) {
    for (int i = 0; i < MAX_PENDING_ACKS; i++) {
        pending_ack_t* e = &table->entries[i];
        if (!e->active) continue;
        if (now_ms >= e->next_retry_ms) {
            e->attempt++;
            if (e->attempt >= e->max_attempts) { e->active = false; continue; }
            uint32_t delay = tier_base_delay_ms(e->tier) << e->attempt;
            e->next_retry_ms = now_ms + jitter_25(delay);
        }
    }
}
```

**逐段解析**：`jitter_25` 给退避加 ±25% 抖动（`base - quarter` 到 `base + quarter`），防止多个待确认包同步重传导致周期性碰撞风暴。`tick` 中 `delay = base << attempt` 是指数退避（每次翻倍），达到 `max_attempts` 即放弃并标记不活跃。

### 5. Fail-Closed 网络密钥与 HMAC（`components/network_key/network_key.c`）

Bramble 的安全基石是"未配钥即静默"——没有网络密钥的节点既不发送也不接收任何认证控制面流量。

```c
static uint8_t s_key[32];
static int s_provisioned = 0;

int network_key_get(uint8_t key_out[32]) {
    if (!s_provisioned)
        return -1; /* fail-closed: no fallback, write nothing to key_out */
    memcpy(key_out, s_key, 32);
    return 0;
}

int network_key_generate_provision(uint8_t key_out[32]) {
    uint8_t key[32];
    if (crypto_random(key, sizeof(key)) != 0)
        return -1; /* entropy gate shut: provision nothing */
    network_key_set_provisioned(key);
    memcpy(key_out, key, sizeof(key));
    return 0;
}
```

**逐段解析**：`network_key_get` 在未配钥时返回 -1 且**不写任何东西到 key_out**——fail-closed，没有零密钥回退。`network_key_generate_provision` 先在临时缓冲 `key` 抽熵，熵门失败则什么都不配（不污染 `key_out`），只有成功才提交。这与 `crypto_generate_identity` 的 fail-closed 模式一致，避免半成品状态。

```c
int network_key_mac(const char* label, const uint8_t* data, size_t len, uint8_t out[8]) {
    uint8_t key[32];
    if (network_key_get(key) != 0) {
        memset(out, 0, 8);
        return -1;
    }
    assert(len <= 255);
    size_t label_len = strlen(label);
    assert(label_len <= NETWORK_KEY_MAC_MAX_LABEL_LEN);
    uint8_t buf[255 + NETWORK_KEY_MAC_MAX_LABEL_LEN];
    memcpy(buf, label, label_len);
    memcpy(buf + label_len, data, len);

    uint8_t full_mac[32];
    crypto_hmac_sha256(key, sizeof(key), buf, label_len + len, full_mac);
    memcpy(out, full_mac, 8);
    return 0;
}
```

**逐段解析（重点）**：这是 wire v4 认证的核心——`label || data` 做 HMAC-SHA256，取前 8 字节作为 MAC。`label` 是域分隔字符串（如 `"bramble-receipt-v2"`），防止不同用途的 MAC 互相伪造（domain separation）。8 字节截断 MAC 在 LoRa 带宽约束下是带宽与安全的折中（64 位碰撞需 2^32 次）。`assert` 保证 label 与 data 不溢出栈缓冲。未配钥时输出全零哨兵并返回 -1，调用方据此拒绝该帧。

```c
void network_key_fingerprint(uint8_t out[4]) {
    uint8_t key[32];
    if (network_key_get(key) != 0) { memset(out, 0, 4); return; }
    uint8_t hash[32];
    crypto_sha256(key, sizeof(key), hash);
    memcpy(out, hash, 4);
}
```

**逐段解析**：网络密钥指纹用于 Web 客户端显示"当前配的是哪个网络"（4 字节，不可逆），让用户能对账确认两端配的是同一把密钥，全零表示未配钥。

### 6. 分组去重与 LRU 淘汰（`components/dedup/dedup.c`）

```c
bool dedup_check_and_add(dedup_buffer_t* buf, uint32_t packet_id, uint32_t now_ms) {
    dedup_purge(buf, now_ms);
    for (int i = 0; i < buf->count; i++) {
        if (buf->entries[i].packet_id == packet_id) return true; /* duplicate */
    }
    if (buf->count < DEDUP_MAX_ENTRIES) {
        buf->entries[buf->count].packet_id = packet_id;
        buf->entries[buf->count].timestamp_ms = now_ms;
        buf->count++;
    } else {
        int oldest = 0;
        for (int i = 1; i < buf->count; i++)
            if (buf->entries[i].timestamp_ms < buf->entries[oldest].timestamp_ms) oldest = i;
        buf->entries[oldest].packet_id = packet_id;
        buf->entries[oldest].timestamp_ms = now_ms;
    }
    return false; /* not duplicate */
}
```

**逐段解析**：先 `purge` 清过期项，再线性查重。表满时淘汰最旧项（LRU by timestamp）。这在泛洪/多跳场景下防止同一 packet_id 被反复中继形成广播风暴。线性查找在 `DEDUP_MAX_ENTRIES` 较小的嵌入式场景下足够，且无哈希碰撞风险。

```c
void dedup_purge(dedup_buffer_t* buf, uint32_t now_ms) {
    int dst = 0;
    for (int src = 0; src < buf->count; src++) {
        if ((now_ms - buf->entries[src].timestamp_ms) < DEDUP_EXPIRY_MS) {
            if (dst != src) buf->entries[dst] = buf->entries[src];
            dst++;
        }
    }
    buf->count = dst;
}
```

**逐段解析**：原地压缩删除——双指针 `dst`/`src`，保留未过期项并前移，最后截断 `count`。O(n) 且无动态分配，适合实时中继路径。

## API 使用指南

Bramble 通过 WebSocket 暴露 JSON-RPC 接口（`main/ws_server.c` + `main/rpc_methods.c`），Web 客户端、桌面应用、CLI 都走同一套 RPC。节点也提供 mDNS 发现与浏览器烧录端点。

### Python：连接节点并发送消息

```python
# pip install websocket-client
import json, websocket

# 节点默认在 80 端口提供 WS 服务（也可经 mDNS 发现 bramble.local）
ws = websocket.create_connection("ws://192.168.4.1/ws")

def call(method, params=None, msg_id=1):
    ws.send(json.dumps({"jsonrpc": "2.0", "method": method,
                        "params": params or {}, "id": msg_id}))
    return json.loads(ws.recv())

# 1. 查询节点状态
print(call("bramble.getStatus"))

# 2. 配置网络密钥（首次配网，32 字节十六进制）
netkey_hex = "a1b2c3...共64个十六进制字符"
print(call("bramble.provisionNetworkKey", {"key": netkey_hex}))

# 3. 发送一条直连消息
print(call("bramble.sendDm", {
    "dest_addr": "1a2b3c4d",      # 目标节点 4 字节地址
    "text": "Hello from Python",
    "tier": "normal",             # broadcast | normal | critical
}))

# 4. 订阅推送通知（GPS、WiFi、新消息等）
ws.send(json.dumps({"jsonrpc": "2.0", "method": "bramble.subscribe",
                    "params": {"events": ["onGpsEvent","onWifiEvent","onMessage"]}, "id": 2}))
while True:
    note = json.loads(ws.recv())
    if "method" in note:          # notification（无 id）
        print("event:", note["method"], note.get("params"))
```

### cURL：经 HTTP 调用节点 REST 端点

```bash
# 查询节点基本信息
curl -s http://192.168.4.1/api/status | jq .

# 触发 OTA 检查（components/ota）
curl -s -X POST http://192.168.4.1/api/ota/check \
     -H "Content-Type: application/json" \
     -d '{"url":"https://releases.example.com/bramble.bin"}'

# 获取邻居与路由表
curl -s http://192.168.4.1/api/mesh/neighbors | jq .

# 浏览器烧录：直接拉取固件镜像
curl -s -o bramble-heltec-v3.bin \
     https://releases.example.com/bramble-heltec-v3.bin
```

> 注：上述端点形态基于 `rpc_methods.c` / `ws_server.c` 的 JSON-RPC over WebSocket 架构与项目 Web 客户端实现；具体方法名以仓库 `docs/` 与 `rpc_methods.h` 为准。

## 编译与部署

### 环境搭建

| 依赖 | 版本 | 说明 |
|------|------|------|
| ESP-IDF | v5.x | 官方开发框架 |
| Python | 3.8+ | IDF 工具链 |
| CMake | 3.16+ | 构建系统 |
| Ninja | 任意 | 推荐构建后端 |
| USB 串口驱动 | 板载 | CP210x / CH340 |

### 编译配置（板级 profile）

| Board Profile | sdkconfig 文件 | 说明 |
|---------------|----------------|------|
| Heltec V3 | `sdkconfig.defaults` | 默认，OLED + SX1262 |
| Heltec V4 | `sdkconfig.defaults.heltec_v4` | + L76K GNSS |
| T-Deck Plus | `sdkconfig.defaults.tdeck_plus` | LCD + 触摸 + 键盘 + 音频 |
| Pager v1 | `sdkconfig.defaults.bramble_pager` | e-paper + GPS |
| 模拟器 | `sdkconfig.defaults.qemu` | Linux host target |

### 烧录步骤

```bash
# 1. 克隆并设置 IDF 环境
git clone https://github.com/justinlindh/bramble.git
cd bramble
. $HOME/esp/esp-idf/export.sh

# 2. 选定目标板（Heltec V3）并编译
idf.py set-target esp32s3
idf.py -DSDKCONFIG_DEFAULTS="sdkconfig.defaults" build

# 3. 烧录（替换为实际串口）
idf.py -p /dev/ttyUSB0 flash monitor

# 4. 首次配网：连接节点 AP 或经 Web 客户端
#    浏览器打开 http://bramble.local 或节点 IP，
#    在 UNPROVISIONED 横幅处生成/粘贴网络密钥
```

也可用项目自带脚本：

```bash
bash scripts/flash.sh local heltec-v3 build
bash scripts/flash.sh local heltec-v3 flash /dev/ttyUSB0
```

无需工具链时，直接用浏览器 Web Flasher 烧录（项目 `web-flasher/` 目录）。

## 项目亮点与适用场景

**亮点**
- **单一 TX 门控路径**：所有发送穿过 `tx_gate`，从架构上保证空口预算与法规占空比不被绕过。
- **诚实的安全边界**：每个特性主张都附带 residual（残余风险），未配钥即静默，无公共默认密钥。
- **host/simulator/emulator 三级验证**：协议逻辑可在主机单测、模拟器规模验证、真实固件二进制仿真中逐级验证后再上设备。
- **自适应预算画像**：空口预算随 mesh 规模自动调整，小 mesh 放大重传、大 mesh 收窄避堵。

**适用场景**
- 应急/野外无基础设施通信（灾区、户外、偏远作业）。
- 隐私敏感的离网消息传递（DM 端到端加密 + 安全码验证）。
- LoRa mesh 协议研究与人门开发（AODV 路由、空口预算、token bucket 的工程参考实现）。
- 多板硬件抽象与 LVGL/e-paper UI 实践。

## 总结

Bramble 是一份少见的"工程严谨度高于其 pre-alpha 标签"的 LoRa mesh 固件。它的价值不在功能多，而在每个特性都把约束想清楚了——空口预算考虑了 ETSI 任意观测窗口的最坏突发，可靠性区分了吞吐与公平，安全用 fail-closed 拒绝半成品状态，路由用双 substrate 把"投递确认"和"路由修复"解耦。对于想深入理解 LoRa 网状网络协议工程化实现的嵌入式开发者，`tx_gate.c`、`airtime_budget.c`、`reliability.c` 这几个文件是非常值得逐行研读的范本。

---

📝 作者：蔡浩宇（jun-chy） ｜ 📅 日期：2026-07-18 ｜ 🔗 项目地址：https://github.com/justinlindh/bramble
