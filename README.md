# 嵌入式日报（Embedded Daily）

> 每日嵌入式开源项目深度解析 · 代码级技术分析
>
> 作者：[蔡浩宇（jun-chy）](https://github.com/jun-chy)

这是一个嵌入式开源项目的深度分析与解读仓库。每天选取 GitHub 上活跃的嵌入式相关开源项目，从硬件架构、固件源码、核心算法到编译部署，进行系统性的代码级技术分析。

## 仓库理念

嵌入式开发不应该是黑盒。这个仓库的目标是：

- **拆解真实项目**：不是浅尝辄止的翻译 README，而是深入源码，提取关键代码片段并逐行解析
- **覆盖主流平台**：ESP32、STM32、RP2040、Arduino、nRF52、嵌入式 Linux 等主流 MCU 平台
- **注重实操**：每篇文章都包含编译方法、烧录步骤、API 调用示例，读完就能上手
- **追踪前沿**：关注近期活跃更新、star 增长快、有实际代码和文档的项目

## 目录结构

```
embedded-daily/
├── README.md                  # 本文件
├── esp32/                     # ESP32 / ESP8266 相关项目
│   └── 2026-06-28-*.md
├── stm32/                     # STM32 相关项目
│   └── 2026-06-29-*.md
├── arduino/                   # Arduino 相关项目
├── rp2040/                    # RP2040 / Raspberry Pi Pico 相关项目
│   └── 2026-06-28-*.md
├── nrf52/                     # Nordic nRF52 相关项目
├── linux-embedded/            # 嵌入式 Linux 相关项目
└── other/                     # 其他嵌入式项目
```

## 已发布文章

### 2026-07-22

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-22 | ESP32 | [NucleoOS](esp32/2026-07-22-nucleoos.md) | 40 | M5Stack Cardputer（ESP32-S3）Web 原生操作系统固件，512KB RAM 无 PSRAM 上实现完整 OS + 离线 AI 助手 ANIMA（L0/L1/L2 检索级联 + Matryoshka 二进制 Hamming popcount + 证据性弃权绝不虚构）+ 三运行时一致性（C 同一份编译为固件/WASM/PC 三目标奇偶校验门）+ 双界面架构（原生 240×135 UI + 浏览器桌面级 Web Shell WebGPU/WASM 重计算卸载）+ Solo 启动模式（RTC 标志跳过 httpd 18KB + mDNS 12KB 获取连续堆）+ 事件溯源总线（环形缓冲 + 双锁分离 SD I/O + NDJSON 日志轮转 + 栈拷贝防 use-after-overwrite）+ 端上 MFCC+DTW 语音命令（延迟加载引用计数 16KB）+ OTA A/B 回滚确认 + BLE 控制器 DRAM 回收 + 启动面包屑追踪 + 50+ 原生应用（游戏/安全实验室/媒体播放器）+ W5500 MACRAW L2/L3 以太网攻击引擎 |

### 2026-07-21

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-21 | ESP32 | [ESP32-Plane-Radar](esp32/2026-07-21-esp32-plane-radar.md) | 791 | ESP32-C3 + 1.28" 圆形 GC9A01 实时 ADS-B 航空雷达固件，adsb.fi HTTPS REST 拉取 + ArduinoJson 流式分块解析（512B 缓冲 + 注入 wifiLoop 回调保活协议栈）+ 平面地球坐标变换（1°≈111km + atan2 方位角 + 像素映射）+ 双缓冲 Sprite 无闪烁渲染（112KB 离屏合成一次性 pushSprite）+ 航向三角符号（sin/cos 旋转向量）+ 速度矢量步进裁剪（0.05t 线性插值逼近圆边界 + 平方距离免开方）+ 量程外飞机边缘方位点设计 + 字段优先级回退容错（true_heading→mag→track→dir）+ alt_baro 联合类型判别 + WiFiManager 强制门户 + NVS 量程/位置/单位持久化 + BOOT 单按钮交互（短按切量程/长按 3s 重置）|

### 2026-07-20

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-20 | ESP32 | [AgentDeck](esp32/2026-07-20-agentdeck.md) | 154 | ESP32 AI 编码代理物理控制面板固件，FreeRTOS 双核架构（Core0 网络/Core1 UI）+ 串行/WiFi 双传输 + 射频停泊省 2.4GHz 带宽 + mDNS 零配置发现守护进程（优先 agent=daemon TXT 记录 + token 分发）+ WebSocket 指数退避重连（3s→8s 封顶）+ 跨核互斥量+环形出队队列线程安全（arduinoWebSockets 非线程安全）+ ArduinoJson 协议解析（state_update/usage_update/sessions_list/timeline）+ UTF-8 截断安全 + 生物态状态机（章鱼/小龙虾/霓虹灯鱼水族箱动画派生）+ 8 板 HAL 编译宏裁剪资源上限 + WiFiManager 非阻塞配网门户 + NTP 配额重置倒计时 + 双 OTA 分区远程升级 + IPS10 ESP-Hosted SDIO 断言规避 |

### 2026-07-18

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-18 | ESP32 | [Bramble](esp32/2026-07-18-bramble-esp32-lora-mesh.md) | 1 | ESP32-S3 隐私优先 LoRa 网状网络固件，SX1262 + ESP-IDF + 双 substrate 路由（反应式 AODV 默认/泛洪可选）+ wire v4 认证流量（network-key HMAC）+ fail-closed 未配钥静默 + 单一 TX 门控路径（真实 ToA 计费 + LBT 先听后说 + beacon 预留）+ 四层 token bucket 空口预算（ETSI 任意窗口 cap/2 合规）+ 三级可靠性（0/3/8 重传 + ±25% 抖动退避）+ Ed25519 加密身份（地址=SHA-256 公钥前 4 字节）+ AES-256-GCM 端到端 DM（X25519 quad-DH + HKDF 棘轮前向保密）+ packet_id LRU 去重 + 离线邮箱 + 三档隐私位置 + 浏览器烧录 |
| 2026-07-18 | STM32 | [Alpha-Flight 2.X](stm32/2026-07-18-alpha-flight-stm32-quadcopter.md) | 1 | STM32F405 四旋翼飞控固件，双 ICM-42688P SPI 冗余 + TIM7 硬件触发 SPI1 DMA 连读（2ms/500Hz）+ DMA 完成中断软件中断触发 PID（NVIC_SetPendingIRQ）+ DSHOT600 四通道 TIM1 DMA Burst 并行输出（CCR1~CCR4 一次 update 写入）+ CRSF 状态机（USART2 DMA 循环 + 追赶指针消费 + 11-bit 通道位流解码 + CRC8 查表）+ 高频级联 PID（微分先行 + 动态反向计算抗饱和）+ 时间片轮询主循环（20ms 磁力计/100ms 安全/500ms 黑匣子）+ 倾角保护 + 低压停机 + IWDG 看门狗 + TPS54560 6S 电源隔离 + SPI 黑匣子临界区防撕裂 |

### 2026-07-17

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-17 | RP2040 | [DSPi](rp2040/2026-07-17-dspi.md) | 1119 | RP2040/RP2350 全功能音频 DSP 固件，USB 声卡 + 板载 DSP 引擎 + 混合 SVF/biquad 参数均衡（110 段）+ BS2B 互补滤波耳机串扰（ITD 全通延时）+ ISO 226:2003 响度补偿双缓冲系数表 + RMS 向上压缩音量均衡器 + 2×N 矩阵混音器 + 双核无锁协作（__dmb/__sev/WFE）+ Q28 定点手写汇编内循环 + PIO/DMA 零 CPU S/PDIF/I2S/PDM 输出 + 10 槽 Flash 预设 + 运行时引脚重分配 |

### 2026-07-16

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-16 | ESP32 | [TARFS](esp32/2026-07-16-tarfs.md) | 5 | MCU 零拷贝只读文件系统，标准 TAR 归档 + mmap 真零拷贝访问 + FNV-1a 哈希二分查找索引（O(log N)）+ 运行时零动态分配 + CRC64/ECMA-182 完整性校验 + 无超级块容错设计 + ESP32 分区内存映射 + VFS 注册 + 每文件仅 24 字节 RAM |
| 2026-07-16 | RP2040 | [GeT_OS](rp2040/2026-07-16-get-os.md) | 3 | RP2040/RP2350 从零手写宏内核，纯汇编上下文切换 + 链表内存分配器 + ELF32 动态加载器（GOT/PLT 重定位 + SysV 哈希符号解析）+ 抢占式多任务调度 + 30+ POSIX 系统调用 + VFS/RAMFS/FAT16 + W5100S TCP/IP + VGA 640×480 + PS/2 键盘 |

### 2026-07-15

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-15 | ESP32 | [OpenC6 BIOS v1.1-ME](esp32/2026-07-15-openc6-bios.md) | 287 | ESP32-C6 开源模块化 BIOS/Payload 固件平台深度更新，ABI 系统调用跳转表（20+ API）+ LP-Core 管理引擎（硬件看门狗 + 智能电源按钮 + SchedUtil 动态调频 + 热保护）+ 日志结构循环文件系统（GC 垃圾回收）+ A/B OTA 防变砖 + PXE 网络启动 + 裸机 Payload 示例 + NVRAM 配置架构 |
| 2026-07-15 | ESP32 | [P4KVM](esp32/2026-07-15-p4kvm.md) | 94 | ESP32-P4 IP KVM 方案，TC358743 HDMI→MIPI CSI-2 桥接 + 1080p RGB888 采集 + 硬件 JPEG 编码 + HTTP multipart/x-mixed-replace MJPEG 流 + WebSocket 键鼠控制协议 + USB HID 分段发送与消息合并 + 以太网 mDNS + HDMI 热插拔自动恢复 |

### 2026-07-14

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-14 | 其他 | [µCNC](other/2026-07-14-ucnc-universal-cnc-firmware.md) | 445 | 通用 CNC 数控固件，AVR/ESP32/STM32/RP2040/SAMD21/LPC176x 多平台 HAL + 6 轴运动控制 + Grbl 协议兼容 + 梯形/S 曲线加减速 + 半角恒等式结点速度优化 + DSS 动态步进过采样 + Bresenham 插补 + 背隙补偿 + 模块事件钩子 + Flash EEPROM 仿真 |

### 2026-07-13

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-13 | ESP32 | [Wi-Fi CSI Presence Radar](esp32/2026-07-13-wifi-presence-detector.md) | 1 | ESP32-C3 Wi-Fi CSI 混合存在雷达，双信号链融合（幅度偏差 + 微多普勒脉冲对）+ 12 项精度增强技术 + Hann 加窗重叠 FFT + CFAR 自适应峰值检测 + 卡尔曼滤波 + Hampel 异常值滤波 + 双 EWMA 呼吸带通 + 相位相干性指数 + NVS 校准持久化 + 静态信道逃逸防死锁 FSM |

### 2026-07-12

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-12 | nRF52 | [InfiniTime](nrf52/2026-07-12-infinitime-pinetime.md) | 3,308 | PineTime 智能手表开源固件，nRF52832 + FreeRTOS + NimBLE BLE 协议栈 + LVGL UI 框架 + ST7789 LCD 驱动 + BMA421 加速度计 + HRS3300 心率传感 + MCUBoot OTA 升级 + NoInit 内存持久化 + GPIOTE 中断消息队列 |
| 2026-07-12 | STM32 | [STM32 Bootloader](stm32/2026-07-12-stm32-bootloader.md) | 1,042 | STM32 可定制 IAP 引导加载器，双 Bank Flash 擦除 + 双字编程 + 硬件 CRC32 校验 + WRP/PCROP/RDP 三级写保护 + 向量表重定位跳转 + 系统存储器跳转 + FatFs SD 卡固件更新 + SCons/IAR/GCC 三工具链支持 |

### 2026-07-11

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-11 | STM32 | [EKF IMU Orientation](stm32/2026-07-11-ekf-imu-orientation.md) | 56 | STM32 6-DOF IMU 扩展卡尔曼滤波器，6 维状态向量（角度+陀螺零偏）+ 雅可比矩阵手动推导 + Joseph 形式协方差更新 + 3×3 伴随矩阵求逆 + Python UART 仿真验证闭环 |

### 2026-07-10

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-10 | 其他 | [Amast](other/2026-07-10-amast.md) | 29 | C99 嵌入式异步框架，Actor 模型 + 层次状态机（HSM）事件驱动内核，LCA 转换算法 + 行为继承 + ISR 安全发布订阅 + 引用计数事件池，FSM 仅 0.88KB / HSM 2.66KB |

### 2026-07-09

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-09 | ESP32 | [CSIght](esp32/2026-07-09-csight-wifi-csi-radar.md) | 142 | Flipper Zero + ESP32 WiFi CSI 穿墙运动雷达，被动嗅探子载波幅度 + 30 帧基线校准 + EMA 自适应滤波（α=0.02）+ 灵敏度阈值检测 + 距离估算 + 多芯片 CSI 配置适配 + 整数三角函数雷达渲染 |

### 2026-07-08

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-08 | ESP32 | [Project Aura AQ](esp32/2026-07-08-project-aura.md) | 683 | ESP32-S3 全功能空气质量监测站，SEN66 多合一传感 + 协议级 I2C 驱动（CRC8/VOC 状态持久化/ASC 校准）+ 两级 Safe Boot 防变砖 + GP8403 DAC 智能通风需求算法 + MQTT/Home Assistant 自动发现 |

### 2026-07-04

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-04 | ESP32 | [esp32-weather-epd](esp32/2026-07-04-esp32-weather-epd.md) | 6,165 | ESP32 超低功耗电子纸天气显示器，14μA 休眠 + 四级电池保护 + 分钟对齐智能休眠 + OpenWeatherMap API + BME280 室内传感 |
| 2026-07-04 | STM32 | [MESC_Firmware](stm32/2026-07-04-mesc-firmware.md) | 257 | STM32 开源 FOC 电机控制固件，全自动参数辨识（R/L/磁链/HFI/死区）+ 多观测器 + 弱磁/MTPA + FreeRTOS + 运行时变量调节 |

### 2026-07-03

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-03 | ESP32 | [Espframe](esp32/2026-07-03-espframe-immich-photo-frame.md) | 182 | ESP32-P4 私有云相框固件，直连 Immich 照片库，加权随机选图 + 3-Slot 幻灯片状态机 + 流式多格式解码（JPEG/PNG/WebP）+ 竖图配对 |
| 2026-07-03 | RP2040 | [Picoware](rp2040/2026-07-03-picoware.md) | 221 | RP2040/RP2350 迷你操作系统固件，视图栈导航 GUI + I2C 南桥驱动 + PIO QSPI PSRAM 扩展（DMA 双通道）+ GameBoy 模拟器 |

### 2026-07-01

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-07-01 | RP2040 | [DSPi](rp2040/2026-07-01-DSPi.md) | 1019 | RP2040/RP2350 全功能音频 DSP 固件，双平台自适应滤波（SVF/Biquad）+ BS2B 交叉馈送 + ISO 226 响度补偿 |
| 2026-07-01 | ESP32 | [OpenC6 BIOS](esp32/2026-07-01-openc6-bios.md) | 266 | ESP32-C6 开源 BIOS 固件平台，ABI 系统调用接口 + A/B OTA 防变砖 + PXE 网络启动 + LP-Core 管理引擎 |

### 2026-06-29

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-06-29 | STM32 | [tinyrtos](stm32/2026-06-29-tinyrtos.md) | 11 | 从零手写的 ARM Cortex-M 抢占式 RTOS 内核，PendSV 上下文切换 + 优先级调度 + HardFault 诊断，内核仅 9KB |
| 2026-06-29 | STM32 | [Filum](stm32/2026-06-29-filum.md) | 28 | MCU 级联邦学习框架，STM32 + LoRa 上的边缘 AI 训练，Top-k 稀疏 Q8 量化 + 高斯差分隐私 + ChaCha20 加密 |

### 2026-06-28

| 日期 | 平台 | 项目 | Stars | 简介 |
|------|------|------|-------|------|
| 2026-06-28 | ESP32 | [Sesame Robot](esp32/2026-06-28-sesame-robot.md) | 3,075 | 基于 ESP32 的开源四足机器人，8 舵机驱动，OLED 表情系统，WiFi 双模式 + RESTful API |
| 2026-06-28 | ESP32 | [OpenC6 BIOS](esp32/2026-06-28-openc6-bios.md) | 238 | 把 PC 主板 BIOS 架构搬进 ESP32-C6，LP-Core 管理引擎 + PXE 网络启动 + A/B OTA |
| 2026-06-28 | RP2040 | [DeskHop](rp2040/2026-06-28-deskhop.md) | 7,624 | 双 RP2040 硬件 KVM 切换器，PIO USB + DMA UART + 绝对坐标鼠标跨屏切换 |
| 2026-06-28 | RP2040 | [DSPi](rp2040/2026-06-28-dspi-audio-dsp.md) | 1,010 | RP2040/RP2350 专业级 USB 音频 DSP，参数均衡 + 矩阵混音 + 房间校正 |

### 文章技术覆盖

每篇文章包含以下内容：

- **项目简介与核心特性**：功能定位、应用场景、特性一览表
- **硬件架构**：系统组成图、引脚配置、硬件清单
- **固件架构**：文件结构表、模块职责、数据流
- **核心代码深度分析**：从源码仓库提取真实代码，逐行/逐段解析
- **编译与部署**：环境搭建、编译配置、烧录步骤、使用方法
- **项目亮点与适用场景**：技术亮点总结、二次开发方向

## 平台说明

| 分类 | 芯片平台 | 说明 |
|------|---------|------|
| `esp32/` | ESP32, ESP32-S2, ESP32-S3, ESP32-C3/C6, ESP8266 | 乐鑫 ESP 系列，WiFi/BLE 集成 |
| `stm32/` | STM32F1/F4/F7/H7, STM32L 系列 | ST Cortex-M 微控制器 |
| `arduino/` | AVR, ESP32 Arduino Core | Arduino 生态项目 |
| `rp2040/` | RP2040, RP2350 | 树莓派 Pico 系列 |
| `nrf52/` | nRF52832, nRF52840, nRF5340 | Nordic 低功耗 BLE |
| `linux-embedded/` | i.MX, Allwinner, Rockchip | 嵌入式 Linux 平台 |
| `other/` | 其他 MCU | PIC、TI MSP430、RISC-V 等 |

## 关于作者

**蔡浩宇（jun-chy）**

嵌入式开发工程师，专注于 MCU 固件开发、硬件驱动和开源项目研究。

- GitHub: [jun-chy](https://github.com/jun-chy)
- 仓库: [embedded-daily](https://github.com/jun-chy/embedded-daily)

## 许可证

文章内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可证。

引用的开源项目代码版权归原作者所有，具体许可证请参考各项目仓库。
