# STM32 Bootloader 深度解析：基于 FAT32 与 SD 卡的 IAP 固件升级方案

> 日期：2026-07-12  
> 分类：STM32  
> Star 数：1042  
> 作者：蔡浩宇（jun-chy）

---

## 一、项目链接信息

| 属性 | 内容 |
|------|------|
| 项目名称 | STM32 Bootloader |
| GitHub 地址 | https://github.com/akospasztor/stm32-bootloader |
| 文档地址 | https://akospasztor.github.io/stm32-bootloader |
| 作者 | Akos Pasztor (akospasztor) |
| Stars | 1042 |
| Forks | 307 |
| 主要语言 | C (98.8%) |
| 许可证 | Apache-2.0 |
| 创建日期 | 2017-04-08 |
| 最近更新 | 2026-07-10 |
| 默认分支 | master |
| 最新版本 | v1.1.3 |
| Topics | boot, bootloader, demo, example, fat32, fatfs, firmware, firmware-updater, flash, flasher, iap, in-app-programming, mcu, microcontroller, microsd, sd, stm32, stm32l4, stm32l476, stm32l496 |

### 项目一句话描述

> Customizable Bootloader for STM32 microcontrollers. This project demonstrates how to perform in-application-programming of a firmware located on external SD card with FAT32 file system.

这是一个面向 STM32 微控制器的可定制 Bootloader，演示了如何通过外置 SD 卡上的 FAT32 文件系统执行 IAP（In-Application Programming，应用内编程）固件升级。它将 Bootloader 与业务应用解耦，让设备在不依赖 J-Link/ST-Link 等烧录器的情况下，仅凭一张 SD 卡即可完成现场固件更新。

---

## 二、项目简介与核心特性

STM32 Bootloader 的设计目标是提供一个工程级、可直接迁移到产品中的 Bootloader 参考实现。它并非简单的"程序跳转"示例，而是覆盖了 Flash 擦写、双 Bank 处理、CRC 完整性校验、写保护配置、系统存储器跳转、向量表重定位等真实产品固件升级所必需的全部要素。

### 核心特性一览表

| 特性 | 说明 |
|------|------|
| IAP 固件升级 | 通过 SD 卡 FAT32 文件系统加载并烧录新固件 |
| 双 Bank Flash 支持 | 针对 STM32L4 系列双 Bank 架构的跨 Bank 擦除 |
| CRC 完整性校验 | 利用硬件 CRC 外设校验固件完整性，防砖 |
| 写保护配置 | 支持通过 Option Bytes 配置 WRP 写保护 |
| 应用跳转 | 支持跳转到用户应用，亦支持跳转到 ST 系统存储器 |
| 向量表重定位 | 跳转前自动设置 `SCB->VTOR`，确保中断向量正确 |
| 跨 Bank 跨页擦除 | 自动计算擦除页数，跨 Bank 1 / Bank 2 连续擦除 |
| 双字编程 | 使用 `FLASH_TYPEPROGRAM_DOUBLEWORD` 提升编程效率 |
| 回读校验 | 编程后立即回读比对，防止硬件错误导致数据损坏 |
| 版本管理 | 通过 major/minor/patch/rc 移位打包成 32 位版本号 |
| 可裁剪配置 | 通过宏开关关闭 CRC、写保护等以适配资源受限场景 |

---

## 三、硬件架构

### 3.1 支持的 MCU 列表

本项目目前针对 STM32L4 系列进行了适配，主控型号如下：

| MCU 型号 | 封装 | Flash | RAM | 评估板 / 硬件 |
|----------|------|-------|-----|----------------|
| STM32L476VG | LQFP100 | 1 MB | 128 KB | Custom HW（自研板） |
| STM32L496VG | LQFP100 | 1 MB | 320 KB | Custom HW（自研板） |
| STM32L496AG | UFBGA169 | 1 MB | 320 KB | 32L496GDISCOVERY 官方探索板 |

STM32L4 系列采用双 Bank Flash 架构（每 Bank 256 页，每页 2 KB），这正是 Bootloader 中"跨 Bank 擦除"逻辑存在的原因。该架构允许在运行应用的同时对另一 Bank 进行擦写，是 1 MB 容量 STM32L4 的关键特性。

### 3.2 Flash 内存映射图

下图以 ASCII 形式展示 STM32L4 1MB Flash 的内存布局，Bootloader 与用户应用在物理地址上严格隔离：

```
        0x0800_0000 +-----------------------+
                    |                       |
                    |   Bootloader (32KB)   |   <-- 启动后首先执行
                    |                       |
        0x0800_7FFF +-----------------------+
                    |                       |
                    |                       |
                    |                       |
                    |   Application Space   |   <-- 用户固件存放区
                    |       (~992KB)        |
                    |                       |
                    |                       |
                    |                       |
        0x080F_FFFB +-----------------------+
                    |   CRC Checksum (4B)   |   <-- 固件完整性校验值
        0x080F_FFFF +-----------------------+

        0x1FFF_0000 +-----------------------+
                    |   System Memory       |   <-- ST 内置 Bootloader
                    |   (ST Bootloader)     |       (UART/USB DFU 等)
                    +-----------------------+
```

要点说明：

- **Bootloader 区**：`0x08000000` 起始，占用 32 KB（16 页），存放 Bootloader 自身代码，启动后由它决定是进入升级流程还是跳转到应用。
- **应用区**：`0x08008000` 起始，到 `0x080FFFFB` 结束，约 992 KB，存放用户业务固件。
- **CRC 区**：`0x080FFFFC` 起始的 4 字节，存放应用固件的 CRC32 校验值，由编译工具链在生成固件时追加。
- **系统存储器**：`0x1FFF0000`，ST 出厂烧录的 Bootloader，支持 UART、USB DFU、I2C、SPI 等多种升级协议，作为兜底的恢复入口。

### 3.3 硬件清单表

| 组件 | 型号 / 规格 | 用途 |
|------|-------------|------|
| 主控 MCU | STM32L476VG / STM32L496VG / STM32L496AG | 运行 Bootloader 与应用 |
| SD 卡接口 | MicroSD / SD（SPI 或 SDIO） | 存放待升级固件 BIN 文件 |
| 文件系统 | FAT32 | 兼容性最好的可写文件系统 |
| 串口 | USART（如 LPUART1/USART3） | 调试日志输出 |
| LED | 若干 GPIO | 升级状态指示 |
| 按键 | GPIO 输入 | 触发升级 / 强制进入 Bootloader |

---

## 四、固件架构

### 4.1 源码目录结构

| 路径 | 说明 |
|------|------|
| `lib/stm32-bootloader/bootloader.h` | Bootloader 公开接口与配置宏声明 |
| `lib/stm32-bootloader/bootloader.c` | Bootloader 核心实现（擦除/编程/校验/跳转） |
| `lib/stm32-bootloader/README.md` | 库使用说明 |
| `lib/FatFs` | FatFs 文件系统中间件（来自 ChaN） |
| `lib/STM32L4xx_HAL_Driver` | ST HAL 库驱动 |
| `lib/CMSIS` | ARM CMSIS 头文件 |
| `src/main.c` | Bootloader 主流程入口（SD 卡读取 + 烧写） |
| `src/system_stm32l4xx.c` | 时钟与系统初始化 |
| `src/stm32l4xx_it.c` | 中断服务函数 |
| `SConstruct` | SCons 构建脚本 |
| `firmware/` | 示例应用工程与升级工具脚本 |
| `tools/` | 辅助脚本（如 Python 打包 CRC） |

### 4.2 模块职责

| 模块 | 职责 |
|------|------|
| Bootloader Core | Flash 擦除、双字编程、CRC 校验、应用跳转、系统存储器跳转、写保护配置 |
| FatFs 层 | 挂载 FAT32 分区，打开并按块读取 SD 卡上的固件 BIN 文件 |
| HAL 层 | 封装底层寄存器操作，提供 `HAL_FLASH_*`、`HAL_FLASHEx_*` API |
| main.c | 业务流程编排：检测 SD 卡 → 打开固件 → 分块读取 → 调用 Bootloader API 烧写 → 校验 → 跳转 |
| 工具脚本 | 在固件 BIN 末尾追加 CRC32，使其与 `CRC_ADDRESS` 处的校验值匹配 |

整体调用关系：`main.c` → `Bootloader_* API` → `HAL_FLASH*` → Flash 控制器寄存器。FatFs 为数据源，HAL 为硬件抽象层，Bootloader Core 居中调度。

---

## 五、核心代码深度分析

本章是全文重点。我们将逐一剖析从源文件中提取的关键代码片段，对每一行进行逐段解析。

### 5.1 bootloader.h 配置宏定义

配置宏是 Bootloader 可裁剪性的核心。源文件 `lib/stm32-bootloader/bootloader.h` 顶部通过一组宏开关控制行为。

```c
/** Select target MCU family */
#define STM32L4

/** Check application checksum on startup */
#define USE_CHECKSUM 0

/** Enable write protection after performing in-app-programming */
#define USE_WRITE_PROTECTION 0

/** Automatically set vector table location before launching application */
#define SET_VECTOR_TABLE 1

/** Clear reset flags */
#define CLEAR_RESET_FLAGS 1

/** Start address of application space in flash */
#define APP_ADDRESS (uint32_t)0x08008000

/** End address of application space (address of last byte) */
#define END_ADDRESS (uint32_t)0x080FFFFB

/** Start address of application checksum in flash */
#define CRC_ADDRESS (uint32_t)0x080FFFFC

/** Address of System Memory (ST Bootloader) */
#define SYSMEM_ADDRESS (uint32_t)0x1FFF0000
```

逐段解析：

- `#define STM32L4`：选择目标 MCU 系列。Bootloader 内部会根据该宏包含对应的 HAL 头文件、使用对应的 Flash 页大小常量。若要移植到 STM32F4/F7/H7，需新增对应宏分支并调整页大小与 Bank 数量。
- `#define USE_CHECKSUM 0`：是否在启动时检查应用 CRC。置 0 表示关闭，置 1 表示开启。关闭时 `Bootloader_VerifyChecksum()` 中的 `#if(USE_CHECKSUM)` 块会被预处理器剔除，函数直接返回 `BL_CHKS_ERROR`。这一设计让用户在工具链不支持自动追加 CRC 时也能使用 Bootloader。
- `#define USE_WRITE_PROTECTION 0`：是否在 IAP 完成后开启写保护。开启后，应用区被 WRP（Write Protection）锁定，防止误擦写。产品环境下建议开启以提升安全性。
- `#define SET_VECTOR_TABLE 1`：跳转到应用前是否自动设置 `SCB->VTOR`。应用的中断向量表必须指向其自身起始地址（`0x08008000`），否则中断会跳到 Bootloader 的向量表导致硬件错误。置 1 表示自动设置。
- `#define CLEAR_RESET_FLAGS 1`：是否清除复位标志。STM32 在 `RCC->CSR` 中保留复位原因（看门狗、软件复位、上电等），清除后可避免应用层误判。
- `#define APP_ADDRESS (uint32_t)0x08008000`：应用区起始地址。这是 Bootloader 与应用的"分水岭"——Bootloader 占用 `0x08000000~0x08007FFF` 共 32 KB，应用从 `0x08008000` 开始。该地址必须与链接脚本中应用的 `FLASH` 段起始地址一致。
- `#define END_ADDRESS (uint32_t)0x080FFFFB`：应用区最后一个字节地址。1 MB Flash 末尾 4 字节留给 CRC，因此应用区结尾为 `0x080FFFFB`（`0x080FFFFC - 1`）。
- `#define CRC_ADDRESS (uint32_t)0x080FFFFC`：CRC 校验值存放地址，紧接应用区末尾，占 4 字节，到 `0x080FFFFF` 结束。
- `#define SYSMEM_ADDRESS (uint32_t)0x1FFF0000`：系统存储器起始地址，ST 出厂 Bootloader 所在位置。`Bootloader_JumpToSysMem()` 跳转到此处，可进入 ST 原生 DFU/UART 升级模式，作为兜底恢复手段。

接下来是错误码与保护类型枚举：

```c
enum eBootloaderErrorCodes
{
    BL_OK = 0,      /*!< No error */
    BL_NO_APP,      /*!< No application found in flash */
    BL_SIZE_ERROR,  /*!< New application is too large for flash */
    BL_CHKS_ERROR,  /*!< Application checksum error */
    BL_ERASE_ERROR, /*!< Flash erase error */
    BL_WRITE_ERROR, /*!< Flash write error */
    BL_OBP_ERROR    /*!< Flash option bytes programming error */
};

enum eFlashProtectionTypes
{
    BL_PROTECTION_NONE  = 0,   /*!< No flash protection */
    BL_PROTECTION_WRP   = 0x1, /*!< Flash write protection */
    BL_PROTECTION_RDP   = 0x2, /*!< Flash read protection */
    BL_PROTECTION_PCROP = 0x4, /*!< Flash propietary code readout protection */
};
```

逐段解析：

- `eBootloaderErrorCodes`：所有 Bootloader API 的统一返回值枚举。`BL_OK = 0` 是惯例，便于用 `if(ret != BL_OK)` 判断失败。每个错误码对应一种失败场景：无应用、尺寸超限、CRC 不符、擦除失败、写入失败、Option Bytes 编程失败。这种"显式错误码"设计比布尔返回值更利于上层定位故障。
- `BL_NO_APP`：当 `Bootloader_CheckForApplication()` 检测到应用起始处的栈指针不在合法 RAM 范围内时返回，表示 Flash 中没有有效应用。
- `BL_SIZE_ERROR`：新固件超出应用区容量时返回，是防砖的第一道防线。
- `eFlashProtectionTypes`：用位掩码定义保护类型。`WRP=0x1`、`RDP=0x2`、`PCROP=0x4`，可用按位或组合。`Bootloader_ConfigProtection(uint32_t protection)` 接收该枚举作为参数，通过 `if(protection & BL_PROTECTION_WRP)` 判断是否启用某项保护。位掩码设计允许未来扩展更多保护类型而不破坏 API 兼容性。

### 5.2 Bootloader_Init() 与 Bootloader_Erase()

`Bootloader_Init()` 负责初始化 Flash 外设，`Bootloader_Erase()` 负责擦除整个应用区。这两者是 IAP 流程的"准备阶段"。

```c
uint8_t Bootloader_Init(void)
{
    __HAL_RCC_SYSCFG_CLK_ENABLE();
    __HAL_RCC_FLASH_CLK_ENABLE();

    /* Clear flash flags */
    HAL_FLASH_Unlock();
    __HAL_FLASH_CLEAR_FLAG(FLASH_FLAG_ALL_ERRORS);
    HAL_FLASH_Lock();

    return BL_OK;
}
```

逐行解析：

- `__HAL_RCC_SYSCFG_CLK_ENABLE();`：开启 SYSCFG（系统配置控制器）时钟。SYSCFG 控制中断向量表重映射、GPIO 复用等，跳转到系统存储器时会用到 `__HAL_SYSCFG_REMAPMEMORY_SYSTEMFLASH()`，因此必须先开时钟。
- `__HAL_RCC_FLASH_CLK_ENABLE();`：开启 Flash 接口时钟。STM32 的 Flash 控制器在 RCC 中有独立时钟门控，必须使能后才能进行擦写操作。
- `HAL_FLASH_Unlock();`：解锁 Flash 控制器。STM32 Flash 默认处于锁定状态（`FLASH->CR` 的 `LOCK` 位），防止误写。`HAL_FLASH_Unlock()` 通过向 `FLASH->KEYR` 写入两把钥匙（`0x45670123` 与 `0xCDEF89AB`）解除锁定。
- `__HAL_FLASH_CLEAR_FLAG(FLASH_FLAG_ALL_ERRORS);`：清除所有 Flash 错误标志。上一次操作可能遗留 `PGSERR`（编程顺序错误）、`PGAERR`（对齐错误）等标志，不清除会阻塞后续操作。
- `HAL_FLASH_Lock();`：重新锁定 Flash。Init 阶段只是清标志，不进行实际写入，故立刻上锁，保持"用完即锁"的安全习惯。
- `return BL_OK;`：Init 不会失败，固定返回成功。

接下来是擦除函数，这是 IAP 中最易踩坑的部分，因为 STM32L4 的双 Bank 架构需要分 Bank 处理：

```c
uint8_t Bootloader_Erase(void)
{
    uint32_t NbrOfPages = 0;
    uint32_t PageError  = 0;
    FLASH_EraseInitTypeDef pEraseInit;
    HAL_StatusTypeDef status = HAL_OK;

    HAL_FLASH_Unlock();

    /* Get the number of pages to erase */
    NbrOfPages = (FLASH_BASE + FLASH_SIZE - APP_ADDRESS) / FLASH_PAGE_SIZE;

    if(NbrOfPages > FLASH_PAGE_NBPERBANK)
    {
        pEraseInit.Banks     = FLASH_BANK_1;
        pEraseInit.NbPages   = NbrOfPages % FLASH_PAGE_NBPERBANK;
        pEraseInit.Page      = FLASH_PAGE_NBPERBANK - pEraseInit.NbPages;
        pEraseInit.TypeErase = FLASH_TYPEERASE_PAGES;
        status               = HAL_FLASHEx_Erase(&pEraseInit, &PageError);

        NbrOfPages = FLASH_PAGE_NBPERBANK;
    }

    if(status == HAL_OK)
    {
        pEraseInit.Banks     = FLASH_BANK_2;
        pEraseInit.NbPages   = NbrOfPages;
        pEraseInit.Page      = FLASH_PAGE_NBPERBANK - pEraseInit.NbPages;
        pEraseInit.TypeErase = FLASH_TYPEERASE_PAGES;
        status               = HAL_FLASHEx_Erase(&pEraseInit, &PageError);
    }

    HAL_FLASH_Lock();

    return (status == HAL_OK) ? BL_OK : BL_ERASE_ERROR;
}
```

逐段解析：

- `uint32_t NbrOfPages = 0;`：待擦除的总页数。STM32L4 每页 2 KB，1 MB Flash 共 512 页，分两个 Bank 各 256 页（`FLASH_PAGE_NBPERBANK`）。
- `uint32_t PageError = 0;`：HAL 擦除失败时回填出错页号，便于定位故障页。
- `FLASH_EraseInitTypeDef pEraseInit;`：HAL 擦除参数结构体，包含 Bank 选择、起始页、页数、擦除类型。
- `HAL_FLASH_Unlock();`：解锁 Flash，准备执行擦除。
- `NbrOfPages = (FLASH_BASE + FLASH_SIZE - APP_ADDRESS) / FLASH_PAGE_SIZE;`：计算从应用起始地址到 Flash 末尾的总页数。代入 STM32L4 1MB 参数：`(0x08000000 + 0x100000 - 0x08008000) / 0x800 = 0xF8000 / 0x800 = 496` 页。注意这里没有为 CRC 区单独留页，因为 CRC 与应用共享最后一页。
- `if(NbrOfPages > FLASH_PAGE_NBPERBANK)`：判断是否需要跨 Bank 擦除。`FLASH_PAGE_NBPERBANK` 为 256，496 > 256，因此进入分支，先擦 Bank 1 中的溢出部分。
- `pEraseInit.Banks = FLASH_BANK_1;`：选择 Bank 1。
- `pEraseInit.NbPages = NbrOfPages % FLASH_PAGE_NBPERBANK;`：`496 % 256 = 240`，即 Bank 1 中需擦除 240 页。这是因为应用区横跨 Bank 1 的尾部和 Bank 2 的全部。
- `pEraseInit.Page = FLASH_PAGE_NBPERBANK - pEraseInit.NbPages;`：`256 - 240 = 16`，即从 Bank 1 的第 16 页（对应地址 `0x08008000`，正好是 `APP_ADDRESS`）开始擦除。这一步巧妙地把"应用起始地址"换算成了"Bank 内页号"。
- `pEraseInit.TypeErase = FLASH_TYPEERASE_PAGES;`：按页擦除而非整片擦除，保护 Bootloader 区域。
- `status = HAL_FLASHEx_Erase(&pEraseInit, &PageError);`：执行 Bank 1 擦除。
- `NbrOfPages = FLASH_PAGE_NBPERBANK;`：将剩余页数重置为 256，表示 Bank 2 全部擦除。
- `if(status == HAL_OK)`：Bank 1 擦除成功才继续擦 Bank 2，避免在已出错状态下继续操作。
- `pEraseInit.Banks = FLASH_BANK_2;`：切换到 Bank 2。
- `pEraseInit.NbPages = NbrOfPages;`：256 页。
- `pEraseInit.Page = FLASH_PAGE_NBPERBANK - pEraseInit.NbPages;`：`256 - 256 = 0`，从 Bank 2 第 0 页开始擦。
- `status = HAL_FLASHEx_Erase(&pEraseInit, &PageError);`：执行 Bank 2 擦除。
- `HAL_FLASH_Lock();`：擦完立刻上锁。
- `return (status == HAL_OK) ? BL_OK : BL_ERASE_ERROR;`：任一 Bank 擦除失败则返回 `BL_ERASE_ERROR`。

这段代码的核心难点在于"跨 Bank 边界的页号换算"。STM32L4 的双 Bank 不是线性连续编址，HAL 的 `Page` 字段是 Bank 内页号（0~255），因此必须先算出 Bank 1 内的偏移页数，再单独处理 Bank 2。这种"分 Bank 处理"的写法是 STM32L4 Bootloader 区别于 STM32F1/F4 单 Bank Bootloader 的关键差异。

### 5.3 Bootloader_FlashBegin / FlashNext / FlashEnd 闪存编程

这三个函数构成一个"流式编程"接口：Begin 初始化指针，Next 每次写入 8 字节，End 收尾。这种设计避免了"一次性把整个固件读入 RAM"的内存压力，非常适合 SD 卡分块读取场景。

```c
static uint32_t flash_ptr = APP_ADDRESS;

uint8_t Bootloader_FlashBegin(void)
{
    flash_ptr = APP_ADDRESS;
    HAL_FLASH_Unlock();
    return BL_OK;
}
```

逐行解析：

- `static uint32_t flash_ptr = APP_ADDRESS;`：模块内静态变量，记录当前写入位置。`static` 限定其作用域为该 .c 文件，外部不可见，保证封装性。初值设为 `APP_ADDRESS`，但 Begin 中会重新赋值，确保多次升级不出错。
- `flash_ptr = APP_ADDRESS;`：每次 Begin 都把指针重置到应用区起始，保证可重复使用。
- `HAL_FLASH_Unlock();`：解锁 Flash，整个编程会话期间保持解锁，直到 End 上锁。
- `return BL_OK;`：Begin 不会失败。

```c
uint8_t Bootloader_FlashNext(uint64_t data)
{
    if(!(flash_ptr <= (FLASH_BASE + FLASH_SIZE - 8)) ||
       (flash_ptr < APP_ADDRESS))
    {
        HAL_FLASH_Lock();
        return BL_WRITE_ERROR;
    }

    if(HAL_FLASH_Program(FLASH_TYPEPROGRAM_DOUBLEWORD, flash_ptr, data) == HAL_OK)
    {
        if(*(uint64_t*)flash_ptr != data)
        {
            HAL_FLASH_Lock();
            return BL_WRITE_ERROR;
        }
        flash_ptr += 8;
    }
    else
    {
        HAL_FLASH_Lock();
        return BL_WRITE_ERROR;
    }

    return BL_OK;
}
```

逐段解析：

- `uint8_t Bootloader_FlashNext(uint64_t data)`：参数为 8 字节（双字）数据。STM32L4 Flash 编程最小单位是双字（64 位），不能单字节写，这是 L4 系列的硬件约束。
- `if(!(flash_ptr <= (FLASH_BASE + FLASH_SIZE - 8)) || (flash_ptr < APP_ADDRESS))`：边界检查。第一个条件确保当前指针 + 8 不超过 Flash 末尾（`-8` 是因为要写 8 字节，留出空间）；第二个条件确保不会写到 Bootloader 区。注意 `||` 与 `!` 的组合：只要"超上限"或"低于下限"任一成立，即越界。这是防砖的关键检查。
- `HAL_FLASH_Lock(); return BL_WRITE_ERROR;`：越界立刻上锁并报错，防止继续写破坏 Bootloader。
- `HAL_FLASH_Program(FLASH_TYPEPROGRAM_DOUBLEWORD, flash_ptr, data)`：调用 HAL 执行双字编程。`FLASH_TYPEPROGRAM_DOUBLEWORD` 指明编程类型。
- `if(... == HAL_OK)`：HAL 返回成功才进行回读校验。
- `if(*(uint64_t*)flash_ptr != data)`：回读校验。把刚写入的地址当作 `uint64_t*` 解引用，与原数据比对。STM32L4 的 Flash 编程有概率因电压、温度等原因失败，回读是工业级的双重保险。
- `flash_ptr += 8;`：指针前进 8 字节，指向下一个待写位置。
- `else` 分支：HAL 编程失败，上锁报错。
- `return BL_OK;`：本次写入成功。

```c
uint8_t Bootloader_FlashEnd(void)
{
    HAL_FLASH_Lock();
    return BL_OK;
}
```

逐行解析：

- `HAL_FLASH_Lock();`：结束编程会话，重新锁定 Flash，恢复安全状态。
- `return BL_OK;`：End 不会失败。

这种"Begin/Next/End"三段式接口的精妙之处在于：上层只需在一个循环里反复调用 `Bootloader_FlashNext()`，每次传入从 SD 卡读到的 8 字节，无需关心 Flash 解锁、地址推进、边界检查等细节。所有危险操作都被封装在 Next 内部，且每次都做了双重校验。

### 5.4 Bootloader_VerifyChecksum() CRC 校验

CRC 校验是固件完整性的最后一道防线。STM32L4 内置硬件 CRC 外设，比软件 CRC 快数倍，且不占用 CPU。

```c
uint8_t Bootloader_VerifyChecksum(void)
{
#if(USE_CHECKSUM)
    CRC_HandleTypeDef CrcHandle;
    volatile uint32_t calculatedCrc = 0;

    __HAL_RCC_CRC_CLK_ENABLE();
    CrcHandle.Instance                     = CRC;
    CrcHandle.Init.DefaultPolynomialUse    = DEFAULT_POLYNOMIAL_ENABLE;
    CrcHandle.Init.DefaultInitValueUse     = DEFAULT_INIT_VALUE_ENABLE;
    CrcHandle.Init.InputDataInversionMode  = CRC_INPUTDATA_INVERSION_NONE;
    CrcHandle.Init.OutputDataInversionMode = CRC_OUTPUTDATA_INVERSION_DISABLE;
    CrcHandle.InputDataFormat              = CRC_INPUTDATA_FORMAT_WORDS;
    if(HAL_CRC_Init(&CrcHandle) != HAL_OK)
    {
        return BL_CHKS_ERROR;
    }

    calculatedCrc = HAL_CRC_Calculate(&CrcHandle, (uint32_t*)APP_ADDRESS, APP_SIZE);

    __HAL_RCC_CRC_FORCE_RESET();
    __HAL_RCC_CRC_RELEASE_RESET();

    if((*(uint32_t*)CRC_ADDRESS) == calculatedCrc)
    {
        return BL_OK;
    }
#endif
    return BL_CHKS_ERROR;
}
```

逐段解析：

- `#if(USE_CHECKSUM)`：整个校验逻辑被宏包裹。若 `USE_CHECKSUM` 为 0，预处理器直接剔除，函数体只剩最后的 `return BL_CHKS_ERROR;`，即"未启用校验则视为校验失败"。这种设计强制上层在关闭校验时必须显式处理该返回值，避免"悄悄跳过校验"。
- `CRC_HandleTypeDef CrcHandle;`：CRC 外设句柄，HAL 风格的标准句柄结构。
- `volatile uint32_t calculatedCrc = 0;`：计算结果。`volatile` 防止编译器优化掉后续的比较（编译器可能认为 CRC 外设寄存器读取值不变而优化掉回读）。
- `__HAL_RCC_CRC_CLK_ENABLE();`：开启 CRC 外设时钟。CRC 默认时钟关闭以省电，使用前必须使能。
- `CrcHandle.Instance = CRC;`：指向 CRC 外设基址寄存器。
- `DefaultPolynomialUse = DEFAULT_POLYNOMIAL_ENABLE`：使用硬件默认多项式（0x04C11DB7，即 CRC-32 标准）。这样无需手动写 `CRC->POL`，与 ST 工具链生成的 CRC 一致。
- `DefaultInitValueUse = DEFAULT_INIT_VALUE_ENABLE`：使用默认初始值 0xFFFFFFFF。
- `InputDataInversionMode = CRC_INPUTDATA_INVERSION_NONE`：输入不反转。
- `OutputDataInversionMode = CRC_OUTPUTDATA_INVERSION_DISABLE`：输出不反转。这四项配置必须与生成固件 CRC 的工具（Python 脚本或 SRecalculate）参数严格一致，否则校验必然失败。
- `InputDataFormat = CRC_INPUTDATA_FORMAT_WORDS`：以 32 位字为单位输入，匹配 Flash 双字编程的天然对齐。
- `HAL_CRC_Init(&CrcHandle)`：初始化 CRC 外设，失败则返回 `BL_CHKS_ERROR`。
- `calculatedCrc = HAL_CRC_Calculate(&CrcHandle, (uint32_t*)APP_ADDRESS, APP_SIZE);`：核心计算。把应用区起始地址当作 `uint32_t*`，从 `APP_ADDRESS` 开始计算 `APP_SIZE` 个字的 CRC。`APP_SIZE` 是应用区字数（不含 CRC 区）。HAL 内部会自动处理 CRC 外设的写入与读取。
- `__HAL_RCC_CRC_FORCE_RESET(); __HAL_RCC_CRC_RELEASE_RESET();`：复位 CRC 外设，清除内部状态，为下次使用准备干净环境。这是好习惯，避免残留状态污染下一次计算。
- `if((*(uint32_t*)CRC_ADDRESS) == calculatedCrc)`：把 `CRC_ADDRESS`（`0x080FFFFC`）处的 4 字节当作 `uint32_t` 读出，与计算值比对。该值由工具链在生成 BIN 时追加。
- `return BL_OK;`：匹配则校验通过。
- 函数末尾 `return BL_CHKS_ERROR;`：无论是 `USE_CHECKSUM=0` 还是校验不匹配，都走到这里返回失败。

注意一个细节：`HAL_CRC_Calculate` 计算的是 `APP_SIZE` 个字，而 `APP_SIZE` 必须是字数（字节数 / 4）。固件打包工具追加 CRC 时，计算的也是同一范围（从应用起始到 CRC 区之前），保证两端对齐。这是 CRC 校验能够成立的基础。

### 5.5 Bootloader_CheckForApplication() 与 Bootloader_JumpToApplication()

这两个函数决定"是否有应用可跳"以及"如何跳"。

```c
uint8_t Bootloader_CheckForApplication(void)
{
    return (((*(uint32_t*)APP_ADDRESS) - RAM_BASE) <= RAM_SIZE) ? BL_OK : BL_NO_APP;
}
```

逐行解析：

- `(*(uint32_t*)APP_ADDRESS)`：读取应用区第一个字。在 ARM Cortex-M 的向量表中，第一个字是初始栈指针（MSP），由链接脚本在编译时填入，必须指向一段合法的 RAM 地址。
- `((*(uint32_t*)APP_ADDRESS) - RAM_BASE)`：栈指针减去 RAM 起始地址，得到栈指针在 RAM 中的偏移量。
- `<= RAM_SIZE`：判断偏移是否在 RAM 范围内。若应用未烧录，该位置通常是 `0xFFFFFFFF`，减去 `RAM_BASE` 后远超 `RAM_SIZE`，返回 `BL_NO_APP`。这是一个巧妙的"应用存在性"启发式检测——无需任何标志位，仅凭栈指针合法性即可判断。
- `? BL_OK : BL_NO_APP`：合法返回 OK，否则返回无应用。

这种检测的可靠性源于：合法应用的栈指针必然落在 RAM 内（如 STM32L476 的 `0x20000000~0x20020000`），而擦除后的 Flash 是 `0xFF`，栈指针 `0xFFFFFFFF` 远超 RAM 范围。比"在固定地址存魔数"更省空间。

```c
void Bootloader_JumpToApplication(void)
{
    uint32_t JumpAddress = *(__IO uint32_t*)(APP_ADDRESS + 4);
    pFunction Jump       = (pFunction)JumpAddress;

    HAL_RCC_DeInit();
    HAL_DeInit();

    SysTick->CTRL = 0;
    SysTick->LOAD = 0;
    SysTick->VAL  = 0;

#if(SET_VECTOR_TABLE)
    SCB->VTOR = APP_ADDRESS;
#endif

    __set_MSP(*(__IO uint32_t*)APP_ADDRESS);
    Jump();
}
```

逐段解析：

- `uint32_t JumpAddress = *(__IO uint32_t*)(APP_ADDRESS + 4);`：读取应用区第二个字（偏移 +4）。在向量表中，第二个字是复位向量（Reset_Handler 地址），即应用启动后第一条执行的指令。`__IO` 是 `volatile` 的 CMSIS 别名，防止编译器优化掉这次读取。
- `pFunction Jump = (pFunction)JumpAddress;`：把该地址强转为函数指针。`pFunction` 定义为 `typedef void (*pFunction)(void);`，即无参无返回的函数指针类型。
- `HAL_RCC_DeInit();`：复位 RCC，恢复时钟到默认状态（HSI），避免应用继承 Bootloader 的时钟配置导致外设异常。
- `HAL_DeInit();`：反初始化所有 HAL 模块，关闭已使能的中断、外设。
- `SysTick->CTRL = 0; SysTick->LOAD = 0; SysTick->VAL = 0;`：停掉 SysTick。Bootloader 用 SysTick 做 HAL 延时，应用会重新配置，必须先清零避免触发未注册的中断。
- `#if(SET_VECTOR_TABLE) SCB->VTOR = APP_ADDRESS; #endif`：把向量表基址寄存器设为 `APP_ADDRESS`。此后所有中断都从应用自己的向量表查找。若不设置，中断会沿用 Bootloader 的向量表，导致应用中断全失效。
- `__set_MSP(*(__IO uint32_t*)APP_ADDRESS);`：设置主栈指针为应用区第一个字（即应用的初始 MSP）。这是跳转前最关键的一步——C 代码运行依赖栈，必须先把栈指针切到应用的栈，否则跳过去后任何函数调用都会栈错乱。
- `Jump();`：调用函数指针，实际是执行一次无条件跳转到应用的 Reset_Handler。从此刻起，CPU 进入应用代码，Bootloader 使命完成。

这 8 行代码浓缩了 Cortex-M 内核"程序跳转"的全部要素：取复位向量、关外设、清 SysTick、重定向向量表、切栈、跳转。任何一步遗漏都会导致应用跑飞。

### 5.6 Bootloader_JumpToSysMem() 跳转到系统存储器

当 SD 卡升级也失败时，可跳转到 ST 出厂 Bootloader，通过 UART/USB DFU 恢复，这是最后的救命稻草。

```c
void Bootloader_JumpToSysMem(void)
{
    uint32_t JumpAddress = *(__IO uint32_t*)(SYSMEM_ADDRESS + 4);
    pFunction Jump       = (pFunction)JumpAddress;

    HAL_RCC_DeInit();
    HAL_DeInit();

    SysTick->CTRL = 0;
    SysTick->LOAD = 0;
    SysTick->VAL  = 0;

    __HAL_RCC_SYSCFG_CLK_ENABLE();
    __HAL_SYSCFG_REMAPMEMORY_SYSTEMFLASH();

    __set_MSP(*(__IO uint32_t*)SYSMEM_ADDRESS);
    Jump();

    while(1)
        ;
}
```

逐段解析：

- `uint32_t JumpAddress = *(__IO uint32_t*)(SYSMEM_ADDRESS + 4);`：从系统存储器 `0x1FFF0000` 读取第二个字，即 ST Bootloader 的复位向量。系统存储器是 ST 出厂烧录的只读区域，内容固定。
- `pFunction Jump = (pFunction)JumpAddress;`：转为函数指针。
- `HAL_RCC_DeInit(); HAL_DeInit();`：与跳应用一样，先还原外设状态。
- `SysTick->CTRL/LOAD/VAL = 0;`：停 SysTick。
- `__HAL_RCC_SYSCFG_CLK_ENABLE();`：开启 SYSCFG 时钟。下一步的内存重映射依赖 SYSCFG，必须先使能其时钟。
- `__HAL_SYSCFG_REMAPMEMORY_SYSTEMFLASH();`：把系统 Flash 映射到 `0x00000000` 起始地址。Cortex-M 复位后默认从 `0x00000000` 取向量，STM32 通过 SYSCFG 的 `MEM_MODE` 位选择映射源（主 Flash / 系统 Flash / SRAM）。此处切换到系统 Flash，确保 ST Bootloader 的向量表被正确访问。
- `__set_MSP(*(__IO uint32_t*)SYSMEM_ADDRESS);`：设置 MSP 为系统存储器第一个字（ST Bootloader 的初始栈）。
- `Jump();`：跳转到 ST Bootloader，进入其升级协议（如 UART Ymodem）。
- `while(1);`：理论上不会执行到这里，但作为防御性编程，万一跳转失败则死循环，防止误执行后续野代码。

注意与 `JumpToApplication` 的差异：跳系统存储器多了 `__HAL_SYSCFG_REMAPMEMORY_SYSTEMFLASH()` 这一步内存重映射，且不需要设置 `SCB->VTOR`（因为重映射后 `0x00000000` 即指向系统 Flash，向量表基址默认就是 0）。这种差异源于 ST Bootloader 与用户应用所处的地址空间不同。

### 5.7 Bootloader_GetVersion() 版本号

```c
#define BOOTLOADER_VERSION_MAJOR 1
#define BOOTLOADER_VERSION_MINOR 1
#define BOOTLOADER_VERSION_PATCH 3
#define BOOTLOADER_VERSION_RC    0

uint32_t Bootloader_GetVersion(void)
{
    return ((BOOTLOADER_VERSION_MAJOR << 24) |
            (BOOTLOADER_VERSION_MINOR << 16) | (BOOTLOADER_VERSION_PATCH << 8) |
            (BOOTLOADER_VERSION_RC));
}
```

逐段解析：

- 四个宏定义版本号分量：major=1、minor=1、patch=3、rc=0，对应 v1.1.3。`RC=0` 表示正式版，非零表示候选版号。
- `BOOTLOADER_VERSION_MAJOR << 24`：major 左移 24 位，占据 bit31~bit24。
- `BOOTLOADER_VERSION_MINOR << 16`：minor 左移 16 位，占据 bit23~bit16。
- `BOOTLOADER_VERSION_PATCH << 8`：patch 左移 8 位，占据 bit15~bit8。
- `(BOOTLOADER_VERSION_RC)`：rc 占据 bit7~bit0。
- `|`：按位或拼接成一个 32 位整数。代入数值：`(1<<24) | (1<<16) | (3<<8) | 0 = 0x01010300`。

这种"移位打包"是嵌入式版本号的经典编码方式，优点是：单个 32 位整数即可表达完整语义版本；上层可用 `(ver >> 24) & 0xFF` 等位运算拆解；便于通过串口、BLE 等窄带通道传输；可直接写入 Flash 固定位用于版本比对。相比字符串 "1.1.3"，节省存储且比较效率高。

### 5.8 Bootloader_ConfigProtection() 写保护配置

写保护通过 Option Bytes 实现，配置后即使代码再次解锁 Flash，被保护页也无法写入，是防误擦的硬件级保险。

```c
uint8_t Bootloader_ConfigProtection(uint32_t protection)
{
    FLASH_OBProgramInitTypeDef OBStruct = {0};
    HAL_StatusTypeDef status            = HAL_ERROR;

    status = HAL_FLASH_Unlock();
    status |= HAL_FLASH_OB_Unlock();

    OBStruct.WRPArea    = OB_WRPAREA_BANK1_AREAA;
    OBStruct.OptionType = OPTIONBYTE_WRP;
    if(protection & BL_PROTECTION_WRP)
    {
        OBStruct.WRPStartOffset = (APP_ADDRESS - FLASH_BASE) / FLASH_PAGE_SIZE;
        OBStruct.WRPEndOffset   = FLASH_PAGE_NBPERBANK - 1;
    }
    else
    {
        OBStruct.WRPStartOffset = 0xFF;
        OBStruct.WRPEndOffset   = 0x00;
    }
    status |= HAL_FLASHEx_OBProgram(&OBStruct);

    if(status == HAL_OK)
    {
        status |= HAL_FLASH_OB_Launch();
    }

    status |= HAL_FLASH_OB_Lock();
    status |= HAL_FLASH_Lock();

    return (status == HAL_OK) ? BL_OK : BL_OBP_ERROR;
}
```

逐段解析：

- `FLASH_OBProgramInitTypeDef OBStruct = {0};`：Option Bytes 编程参数结构体，零初始化确保所有字段有确定值。
- `HAL_StatusTypeDef status = HAL_ERROR;`：状态变量，初值设为错误以便 `|=` 累积。
- `status = HAL_FLASH_Unlock();`：解锁 Flash 主锁。
- `status |= HAL_FLASH_OB_Unlock();`：再解锁 Option Bytes 锁。STM32 的 Option Bytes 有独立于 Flash 主锁的第二把锁（`FLASH->OPTKEYR`），必须双重解锁。
- `OBStruct.WRPArea = OB_WRPAREA_BANK1_AREAA;`：选择 WRP 区域。STM32L4 每个 Bank 有 A/B 两个 WRP 区域，这里选 Bank1 的 Area A。
- `OBStruct.OptionType = OPTIONBYTE_WRP;`：操作类型为 WRP（写保护），而非 RDP 或 PCROP。
- `if(protection & BL_PROTECTION_WRP)`：按位与判断是否要求启用 WRP。这呼应了枚举的位掩码设计——可同时传入多种保护类型，函数按位检测。
- `OBStruct.WRPStartOffset = (APP_ADDRESS - FLASH_BASE) / FLASH_PAGE_SIZE;`：计算保护起始页号。`(0x08008000 - 0x08000000) / 0x800 = 0x8000 / 0x800 = 16`，即从第 16 页（应用区起始）开始保护。前 16 页（Bootloader）不保护。
- `OBStruct.WRPEndOffset = FLASH_PAGE_NBPERBANK - 1;`：`256 - 1 = 255`，保护到 Bank1 最后一页。注意 WRP 是按 Bank 划分的，此处只配置了 Bank1 的保护范围，覆盖了应用区在 Bank1 的部分。
- `else` 分支：若关闭 WRP，则把起始偏移设为 `0xFF`、结束设为 `0x00`，由于 `StartOffset > EndOffset`，HAL 会判定为"无保护区域"，从而清除已有保护。这是 STM32 取消 WRP 的标准写法。
- `status |= HAL_FLASHEx_OBProgram(&OBStruct);`：把配置写入 Option Bytes 寄存器。此时只是写入待生效值，并未真正应用。
- `if(status == HAL_OK) status |= HAL_FLASH_OB_Launch();`：调用 `OB_Launch` 触发 Option Bytes 重载。STM32 的 OB 修改需要触发一次 BOR（Brown-Out Reset）级重载才能生效，`OB_Launch` 内部执行该重载，会引发一次系统复位。
- `status |= HAL_FLASH_OB_Lock(); status |= HAL_FLASH_Lock();`：依次锁定 OB 锁与 Flash 主锁，恢复保护状态。
- `return (status == HAL_OK) ? BL_OK : BL_OBP_ERROR;`：任一步失败返回 `BL_OBP_ERROR`。

关键认知：调用 `HAL_FLASH_OB_Launch()` 会触发系统复位，因此该函数返回后程序实际会重启，后续代码不会执行。这是 OB 编程的特殊性，必须在设计调用流程时考虑——通常在 IAP 全部完成、最后一刻才调用此函数开启保护。

---

## 六、API 使用指南

### 6.1 IAP 编程流程

完整的 IAP 升级流程严格按以下顺序调用 API，任何顺序错乱都可能导致砖机：

| 步骤 | API 调用 | 作用 |
|------|----------|------|
| 1 | `Bootloader_GetProtectionStatus()` | 检查当前写保护状态 |
| 2 | `Bootloader_ConfigProtection(BL_PROTECTION_NONE)` | 禁用写保护，允许擦写 |
| 3 | `Bootloader_Init()` | 初始化 Flash 外设，清错误标志 |
| 4 | `Bootloader_Erase()` | 擦除整个应用区（跨 Bank） |
| 5 | `Bootloader_FlashBegin()` | 解锁 Flash，重置写入指针 |
| 6 | `Bootloader_FlashNext(data)` | 循环调用，每次写入 8 字节 |
| 7 | `Bootloader_FlashEnd()` | 锁定 Flash，结束编程会话 |
| 8 | `Bootloader_VerifyChecksum()` | 校验 CRC32，验证固件完整性 |
| 9 | `Bootloader_JumpToApplication()` | 跳转到新应用 |

典型 main.c 编排伪代码：

```c
/* main.c 中的升级主流程（示意） */
FRESULT fr;
FIL firmwareFile;
uint8_t ret;

ret = Bootloader_Init();
if(ret != BL_OK) Error_Handler();

/* 打开 SD 卡上的固件 */
fr = f_open(&firmwareFile, "firmware.bin", FA_READ);
if(fr != FR_OK) { /* 无固件，直接跳转到现有应用 */ Bootloader_JumpToApplication(); }

/* 擦除 */
if(Bootloader_Erase() != BL_OK) Error_Handler();

/* 流式写入 */
Bootloader_FlashBegin();
UINT bytesRead;
uint8_t buf[8];
do {
    f_read(&firmwareFile, buf, 8, &bytesRead);
    if(bytesRead == 8) {
        if(Bootloader_FlashNext(*(uint64_t*)buf) != BL_OK) {
            Bootloader_FlashEnd();
            Error_Handler();
        }
    }
} while(bytesRead == 8);
Bootloader_FlashEnd();
f_close(&firmwareFile);

/* 校验并跳转 */
if(Bootloader_VerifyChecksum() == BL_OK) {
    Bootloader_JumpToApplication();
}
Error_Handler();
```

流程要点：擦除前必须确保写保护已解除；写入循环每次严格 8 字节；CRC 校验通过才允许跳转，否则保留 Bootloader 等待下一次升级。

### 6.2 Python 脚本示例：为固件追加 CRC

固件 BIN 末尾的 CRC 必须与 Bootloader 计算方式一致。以下 Python 脚本使用与 STM32 硬件 CRC 相同的参数（多项式 0x04C11DB7、初值 0xFFFFFFFF、不反转）：

```python
#!/usr/bin/env python3
# firmware_pack.py - 为固件 BIN 追加 CRC32，匹配 STM32 硬件 CRC
import struct
import sys

def stm32_crc32(data):
    """模拟 STM32 硬件 CRC：多项式 0x04C11DB7，初值 0xFFFFFFFF，无输入输出反转"""
    crc = 0xFFFFFFFF
    # 按 32 位字处理
    for i in range(0, len(data), 4):
        word = struct.unpack('<I', data[i:i+4].ljust(4, b'\xff'))[0]
        crc ^= word
        for _ in range(32):
            if crc & 0x80000000:
                crc = ((crc << 1) ^ 0x04C11DB7) & 0xFFFFFFFF
            else:
                crc = (crc << 1) & 0xFFFFFFFF
    return crc

def main():
    if len(sys.argv) != 3:
        print("Usage: python3 firmware_pack.py input.bin output.bin")
        sys.exit(1)
    with open(sys.argv[1], 'rb') as f:
        firmware = f.read()
    # 计算 CRC（覆盖整个固件区）
    crc = stm32_crc32(firmware)
    # 追加 4 字节小端序 CRC
    with open(sys.argv[2], 'wb') as f:
        f.write(firmware)
        f.write(struct.pack('<I', crc))
    print(f"CRC = 0x{crc:08X}, output: {sys.argv[2]}")

if __name__ == '__main__':
    main()
```

脚本说明：

- `stm32_crc32` 函数严格复现 STM32 硬件 CRC 行为：32 位字输入、多项式 `0x04C11DB7`、初值 `0xFFFFFFFF`、输入输出均不反转。这与 Bootloader 中 `DEFAULT_POLYNOMIAL_ENABLE`、`DEFAULT_INIT_VALUE_ENABLE`、`CRC_INPUTDATA_INVERSION_NONE`、`CRC_OUTPUTDATA_INVERSION_DISABLE` 完全对应。
- `struct.unpack('<I', ...)`：小端序读取 4 字节为无符号 32 位整数，匹配 STM32 的 ARM 小端架构。
- `crc ^= word`：先把当前字异或到 CRC 高位，再循环 32 次移位。
- 末尾 `f.write(struct.pack('<I', crc))`：把 4 字节 CRC 以小端序追加到 BIN 末尾，正好落在 `0x080FFFFC`（即 `CRC_ADDRESS`）处。

执行：`python3 firmware_pack.py app.bin app_crc.bin`，将 `app_crc.bin` 拷贝到 SD 卡即可。

### 6.3 cURL 示例：从 OTA 服务器下载固件

若产品接入网络，可先通过 cURL/HTTP 拉取固件到 SD 卡再升级：

```bash
# 1. 通过 WiFi/以太网模块从 OTA 服务器下载固件到 SD 卡
curl -L -o /sdcard/firmware.bin \
     -H "Authorization: Bearer <TOKEN>" \
     https://ota.example.com/firmware/v1.1.3/app_crc.bin

# 2. 校验文件大小（可选）
ls -l /sdcard/firmware.bin

# 3. 计算 MD5 比对服务器公布的哈希
md5sum /sdcard/firmware.bin

# 4. MCU 端检测到 firmware.bin 存在，触发 IAP 流程
#    （MCU 通过 SPI/SDIO 读取 SD 卡，调用 Bootloader_* API 完成烧写）
```

实际嵌入式场景中，cURL 在主机侧执行；MCU 侧若带网络栈，可用 HTTP 客户端库（如 mbedTLS + HTTP）替代，下载完成后写入 SD 卡或直接送入 `Bootloader_FlashNext()` 流式编程。

---

## 七、编译与部署

### 7.1 SCons 构建

项目原生支持 SCons 构建系统，`SConstruct` 中定义了工具链、源文件列表、编译选项：

```bash
# 安装 SCons 与 ARM GCC 工具链
sudo apt install scons gcc-arm-none-eabi

# 进入项目根目录编译 Bootloader
cd stm32-bootloader
scons -j4

# 生成 build/bootloader.bin / build/bootloader.elf / build/bootloader.hex
ls -l build/
```

SCons 的优势是 Python 语法，比 Makefile 更易维护依赖关系，适合跨平台团队。

### 7.2 IAR EWARM

工程提供 `.eww`/`.ewp` IAR 工程文件，可直接用 IAR 打开：

1. 启动 IAR EWARM for ARM。
2. `File → Open → Workspace`，选择 `stm32-bootloader.eww`。
3. 在工程选项中确认目标芯片为 STM32L476VG / STM32L496VG / STM32L496AG。
4. `Project → Rebuild All`。
5. 通过 ST-Link 下载 `bootloader.out`。

### 7.3 ARM GCC 工具链

无 IDE 环境下可手动调用 arm-none-eabi-gcc：

```bash
# 编译
arm-none-eabi-gcc -c -mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard \
    -DSTM32L476xx -DUSE_HAL_DRIVER -O2 -Wall \
    -Ilib/CMSIS -Ilib/STM32L4xx_HAL_Driver/Inc \
    src/main.c lib/stm32-bootloader/bootloader.c -o build/

# 链接
arm-none-eabi-gcc -T STM32L476VGTX_FLASH.ld -mcpu=cortex-m4 -mthumb \
    -Wl,--gc-sections build/*.o -o build/bootloader.elf

# 转 hex / bin
arm-none-eabi-objcopy -O ihex build/bootloader.elf build/bootloader.hex
arm-none-eabi-objcopy -O binary -S build/bootloader.elf build/bootloader.bin
```

链接脚本（`.ld`）中必须把 `FLASH` 段起始设为 `0x08000000`，长度 32 KB，保证 Bootloader 自身落在保护区。

### 7.4 烧录步骤

| 步骤 | 操作 | 工具 |
|------|------|------|
| 1 | 烧录 Bootloader 到 `0x08000000` | ST-Link Utility / OpenOCD / J-Flash |
| 2 | 编译应用，链接脚本起始设为 `0x08008000` | 任意工具链 |
| 3 | 用 Python 脚本为应用 BIN 追加 CRC | `firmware_pack.py` |
| 4 | 将带 CRC 的 BIN 拷贝到 SD 卡根目录 | 读卡器 |
| 5 | 上电，Bootloader 检测 SD 卡，执行 IAP | 设备自身 |
| 6 | 校验通过后自动跳转到新应用 | Bootloader |

OpenOCD 烧录 Bootloader 示例：

```bash
openocd -f interface/stlink.cfg -f target/stm32l4x.cfg \
        -c "program build/bootloader.hex verify reset exit"
```

---

## 八、项目亮点与适用场景

### 8.1 项目亮点

1. **工程级代码质量**：所有 API 都有边界检查、回读校验、错误码返回，远超"能跑就行"的教学示例，可直接用于量产产品。
2. **双 Bank 架构正确处理**：`Bootloader_Erase()` 的跨 Bank 擦除逻辑是同类项目中少见的正确实现，多数示例仅支持单 Bank。
3. **可裁剪配置**：通过宏开关关闭 CRC、写保护，适配从最小资源到全功能的各类需求。
4. **完整的恢复链路**：SD 卡升级失败可跳转 ST 系统存储器走 DFU，三级恢复保障（应用 → SD IAP → ST Bootloader）。
5. **流式编程接口**：Begin/Next/End 三段式设计，天然契合 SD 卡分块读取，RAM 占用极低。
6. **完善的文档与示例**：配套 GitHub Pages 文档、示例应用工程、Python 打包脚本，开箱即用。

### 8.2 适用场景

| 场景 | 适配性 | 说明 |
|------|--------|------|
| 现场设备固件升级 | 极佳 | 维护人员只需 SD 卡，无需烧录器 |
| OTA 备用通道 | 良好 | 网络升级失败时，SD 卡作为本地兜底 |
| 量产烧录 | 一般 | 量产建议用 ST-Link 直烧，Bootloader 主要服务售后 |
| 远程无人值守升级 | 良好 | 配合通信模块可拉取固件后写入 SD 触发升级 |
| 教学 / 研究 | 极佳 | 代码结构清晰，是学习 IAP/Bootloader 的优秀范本 |
| 安全启动 | 需扩展 | 原生仅 CRC 完整性，签名验证需自行追加（如 RSA/ECDSA） |

---

## 九、总结

STM32 Bootloader 是一个"小而美"的工程级 Bootloader 参考实现。它没有炫酷的功能堆砌，却在每一个细节上做到位：双 Bank 擦除的正确处理、双字编程的边界检查、硬件 CRC 的高效利用、跳转前的栈与向量表设置、写保护的硬件级保障。这些细节正是把"能跑的 Demo"变成"能量产的产品"的分水岭。

从代码看作者的设计哲学：

- **防御式编程**：每一次 Flash 操作都做边界检查与回读校验，把"硬件可能出错"作为前提。
- **显式错误码**：拒绝布尔返回，让上层能精确定位失败环节。
- **可裁剪配置**：宏开关让同一份代码适配从精简到全功能的多种部署。
- **流式接口**：Begin/Next/End 把"大固件写入"问题降维为"循环喂 8 字节"，内存友好。

对于正在做 STM32 产品固件升级方案的开发者，这个项目值得逐行研读。它给出的不仅是一套 API，更是一套"如何安全地操作 Flash"的方法论：什么时候解锁、什么时候上锁、什么时候校验、什么时候跳转，每一步都有其工程必然性。理解了这些，移植到 STM32F4/F7/H7/G4 等其他系列只是修改页大小与 Bank 数量的事，核心思路完全通用。

---

📝 作者：蔡浩宇（jun-chy）  
📅 日期：2026-07-12  
🔗 项目地址：https://github.com/akospasztor/stm32-bootloader
