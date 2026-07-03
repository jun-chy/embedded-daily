# Espframe：ESP32-P4 私有云相框固件深度解析

> 📅 日期：2026-07-03  
> 📂 分类：ESP32  
> ⭐ Star 数：182  
> ✍️ 作者：蔡浩宇（jun-chy）

---

## 项目链接

| 属性 | 值 |
|------|------|
| GitHub 地址 | [jtenniswood/espframe](https://github.com/jtenniswood/espframe) |
| 作者 | jtenniswood |
| 许可证 | PolyForm Noncommercial License 1.0.0 |
| 最近更新 | 2026-07-03 |
| 主要语言 | C++（ESPHome 组件 + YAML 配置） |
| 创建时间 | 2026-03-09 |
| Fork 数 | 13 |
| Topics | esp32, esphome, immich |
| 官方文档 | [jtenniswood.github.io/espframe](https://jtenniswood.github.io/espframe/) |

---

## 项目简介与核心特性一览

Espframe 是一款把 Guition ESP32-P4 触摸屏变成**私有云数字相框**的固件项目。它直连你自建的 [Immich](https://immich.app/) 照片库，无需平板、树莓派、Home Assistant 或任何云订阅——浏览器刷入固件、连 WiFi、填入 Immich 地址和 API Key，即可开始展示照片。所有照片都在你自己的网络中流转，不存在 Espframe 云服务。

项目核心价值在于：在一个内存受限的 MCU 上，完整实现了照片搜索（支持相册/人物/标签/收藏/「那年今日」）、幻灯片调度、多格式图片解码（JPEG/PNG/WebP/BMP）、竖图配对显示、屏幕色调调节与亮度调度，并提供了触摸手势控制和 Web 设置页面。

| 特性 | 说明 |
|------|------|
| Immich 直连 | 直连自建 Immich 服务器，无第三方云 |
| 多照片源 | 全库/收藏/相册/人物/标签/那年今日/日期范围 |
| 竖图配对 | 同日竖图并排显示，宽屏不再大片留白 |
| 多格式解码 | JPEG（libjpeg-turbo）、PNG、WebP、BMP |
| 屏幕调节 | 亮度、暖色调、日落后柔光、定时关屏 |
| 触摸控制 | 单击唤醒、双击下一张、长按休眠 |
| Web 设置 | 手机/电脑浏览器修改全部参数 |
| Home Assistant | 可选接入，作为 ESPHome 设备 |
| 浏览器刷写 | Web Installer 免装开发工具 |
| 条件请求 | ETag / Last-Modified 304 跳过重复下载 |

---

## 硬件架构

### 系统组成图

```
┌─────────────────────────────────────────────────────────────────┐
│              Guition ESP32-P4 (JC8012P4A1) 10" 屏               │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                 ESP32-P4 SoC                               │  │
│  │                                                           │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐  │  │
│  │  │ ESPHome    │->│ Espframe   │->│ Immich HTTP API    │  │  │
│  │  │ Core       │  │ Component  │  │ (search/random)    │  │  │
│  │  └────────────┘  └─────┬──────┘  └────────────────────┘  │  │
│  │                        │                                  │  │
│  │  ┌─────────────────────V──────────────────────────────┐  │  │
│  │  │            Slideshow Controller                     │  │  │
│  │  │  3-Slot 环形缓冲 + FetchQueue 优先级队列              │  │  │
│  │  │  竖图配对状态机 + 预取调度                            │  │  │
│  │  └─────────────────────┬──────────────────────────────┘  │  │
│  │                        │                                  │  │
│  │  ┌─────────────────────V──────────────────────────────┐  │  │
│  │  │            remote_image 组件                         │  │  │
│  │  │  HTTP 流式下载 + 魔数格式检测 + 解码器选择            │  │  │
│  │  │  JPEG / PNG / WebP / BMP  -> RGB565 像素缓冲         │  │  │
│  │  └─────────────────────┬──────────────────────────────┘  │  │
│  │                        │                                  │  │
│  │  ┌──────────┐  ┌───────V────────┐  ┌──────────────────┐ │  │
│  │  │ LVGL UI  │  | GSL3680 触摸   │  | MIPI-DSI 800x480  | │  │
│  │  | 幻灯片屏 |  | (I2C 多点触控)  |  | IPS 显示面板       | │  │
│  │  └──────────┘  └────────────────┘  └──────────────────┘ │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
│  │ USB-C    │  │ WiFi     │  │ 背光 PWM │   <- 可选 ESP32-S3    │
│  │ (刷写)   │  │ (配网)   │  │ (亮度)   │      WiFi 模组         │
│  └──────────┘  └──────────┘  └──────────┘                      │
└─────────────────────────────────────────────────────────────────┘
         │
         │  局域网 HTTPS
         ▼
┌─────────────────────┐
│   Immich 服务器      │  (自建照片库，提供 /api/search/random)
│   /serve-api 原图    │
└─────────────────────┘
```

### 硬件清单

| 组件 | 型号 / 说明 |
|------|------------|
| 主控 | ESP32-P4（双核 RISC-V，800MHz，无内置 WiFi，需外挂 S3） |
| 显示屏 | 10" Guition JC8012P4A1，1024×600 IPS 触摸屏 |
| 触摸控制器 | GSL3680（I2C 多点电容触摸） |
| 无线 | 外挂 ESP32-S3 提供 WiFi/BLE |
| 供电 | USB-C（需数据线） |
| 支架 | 3D 打印 10" 支架（MakerWorld 模型） |

### 触摸手势定义

| 手势 | 动作 |
|------|------|
| 单击 | 唤醒屏幕 |
| 双击 | 切换到下一张照片 |
| 长按 | 休眠 |

---

## 固件架构

Espframe 基于 ESPHome 构建，采用「YAML 声明式配置 + C++ 自定义组件」的混合架构。核心逻辑封装在 `components/espframe/` 和 `components/remote_image/` 两个自定义组件中。

### 文件结构与模块职责

| 路径 | 模块 | 职责 |
|------|------|------|
| `components/espframe/espframe_component.h` | 主组件 | 封装 EspFrameSlideshow，注册到 ESPHome |
| `components/espframe/slideshow_controller.h` | 幻灯片状态机 | 下载完成处理、预取调度、竖图配对决策 |
| `components/espframe/slideshow_component.h` | 幻灯片运行时 | 3-Slot 缓冲、计时、API 调用编排 |
| `components/espframe/immich_helpers.h` | Immich API 工具 | 搜索请求体构建、日期范围解析、UUID 处理、资产解析 |
| `components/espframe/espframe_helpers.h` | 通用辅助 | Slot/Display 状态结构、辅助函数 |
| `components/espframe/sun_calc.h` | 日出日落 | 计算日落时间用于暖光切换 |
| `components/espframe/date_utils.h` | 日期工具 | 公历日期 ⇄ 天数转换 |
| `components/remote_image/remote_image.cpp` | 远程图像 | HTTP 流式下载、格式检测、解码调度、像素缓冲 |
| `components/remote_image/jpeg_image.cpp` | JPEG 解码 | libjpeg-turbo 解码到 RGB565 |
| `components/remote_image/webp_image.cpp` | WebP 解码 | libwebp 解码 |
| `components/gsl3680/gsl3680.cpp` | 触摸驱动 | GSL3680 I2C 触摸驱动 |
| `components/libjpeg-turbo-esp32/` | JPEG 库 | ESP32 优化的 libjpeg-turbo 移植 |
| `common/addon/immich_api.yaml` | API 编排 | ESPHome HTTP 请求 + JSON 解析 lambda |
| `common/addon/immich_slideshow.yaml` | 幻灯片逻辑 | 计时器、轮播、竖图配对触发 |
| `builds/guition-esp32-p4-*.yaml` | 构建配置 | 针对特定硬件的工厂固件配置 |

### 数据流

```
Immich /api/search/random  ──>  HTTP 响应(JSON)
        │
        ▼
  immich_helpers.h          解析 JSON，提取 asset_id/image_url/date/orientation
        │
        ▼
  SlideshowController       决策：显示当前 / 启动竖图对 / 预取下两张
        │
        ▼
  FetchQueue(优先级队列)     按优先级弹出下载任务
        │
        ▼
  remote_image::OnlineImage HTTP 流式下载 + 魔数检测 + 解码
        │
        ▼
  RGB565 像素缓冲 ──> LVGL Image ──> MIPI-DSI 显示
```

---

## 核心代码深度分析

### 一、Immich API 搜索请求构建（immich_helpers.h）

Espframe 需要向 Immich 的 `/api/search/random` 端点发起 POST 请求来获取随机照片。由于 ESPHome lambda 环境不便引入完整的 JSON 库，作者选择**手工拼接 JSON 字符串**，既保持 header-only 可用性，又避免额外依赖。

```cpp
inline std::string build_immich_search_body(int size, bool with_people,
                                             const std::string &photo_source,
                                             const std::string &album_ids,
                                             const std::string &person_ids,
                                             const std::string &tag_ids,
                                             const std::string &extra = "") {
  // 手工拼接 JSON，保持 header-only 可在 ESPHome lambda 中使用
  std::string body = "{\"size\":" + std::to_string(size) +
                      ",\"type\":\"IMAGE\",\"visibility\":\"timeline\",\"withExif\":true";
  if (with_people) body += ",\"withPeople\":true";
  if (!extra.empty()) body += "," + extra;          // 注入 takenAfter/takenBefore 等额外字段
  if (photo_source == "Favorites") {
    body += ",\"isFavorite\":true";                 // 收藏照片源
  } else if (photo_source == "Album" && !album_ids.empty()) {
    body += ",\"albumIds\":" + build_uuid_json_array(album_ids);  // 指定相册
  } else if (photo_source == "Person" && !person_ids.empty()) {
    std::string one = pick_one_person_id_for_random_search(person_ids);
    if (!one.empty())
      body += ",\"personIds\":" + build_uuid_json_array(one);     // 随机选一个人物
  } else if (photo_source == "Tag" && !tag_ids.empty()) {
    body += ",\"tagIds\":" + build_uuid_json_array(tag_ids);      // 指定标签
  }
  body += "}";
  return body;
}
```

**逐段解析：**

- `size` 控制单次请求返回的照片数量，幻灯片预取时通常请求若干张填满 3 个 slot。
- `type:IMAGE` + `visibility:timeline` 限定只返回时间线中的图片资产，过滤掉视频和隐藏内容。
- `withExif:true` 要求返回 EXIF 信息，用于提取拍摄日期和位置；`withPeople` 可选带回人物信息。
- `extra` 参数是关键的扩展点：日期范围过滤（「那年今日」/自定义日期段）通过它注入 `takenAfter` / `takenBefore` 字段，这样主函数无需关心日期逻辑。
- **人物源的特殊处理**：Immich 对多个 `personIds` 采用 AND 语义（照片必须包含所有人）。为了让相框能轮换展示不同人物，作者在 `pick_one_person_id_for_random_search()` 中用 `esp_random()` **随机抽取一个** UUID 发送，长期来看就是「any-of」效果。这是一个针对 API 语义限制的精巧设计。

### 二、随机照片选择——加权时间线桶（immich_helpers.h）

当照片源为相册/人物/标签时，Espframe 不会简单随机翻页，而是先查询时间线「桶」（按月或按天分组），再**按桶内照片数量加权随机**选取一个桶，最后在桶内随机挑一张。这避免了「某月照片多但被均匀随机选中的概率和少照片月份一样」的不公平。

```cpp
inline ImmichTimelineBucketChoice pick_immich_timeline_bucket_from_choices(
    const std::vector<ImmichTimelineBucketInfo> &buckets,
    uint16_t page_size = IMMICH_ALBUM_PAGE_SIZE) {
  std::vector<ImmichTimelineBucketInfo> choices;
  uint32_t total = 0;
  for (const auto &bucket : buckets) {
    if (bucket.time_bucket.empty()) continue;
    uint32_t count = bucket.count == 0 ? 1 : bucket.count;  // 空桶至少算 1，避免除零
    choices.push_back({bucket.time_bucket, count});
    total += count;
  }
  if (choices.empty() || total == 0) return {};
  uint32_t pick = esp_random() % total;            // 在 [0, total) 范围投一个随机数
  uint32_t seen = 0;
  for (const auto &choice : choices) {
    seen += choice.count;                          // 累加权重
    if (pick < seen) {                             // 落入当前桶的区间
      return {choice.time_bucket, choice.count,
              immich_album_page_for_count(choice.count, page_size)};  // 计算该桶分页
    }
  }
  const auto &choice = choices.back();
  return {choice.time_bucket, choice.count,
          immich_album_page_for_count(choice.count, page_size)};
}
```

**逐段解析：**

- `total` 累加所有桶的照片数，构成一个「权重区间」。
- `esp_random() % total` 产生一个落在 `[0, total)` 的随机数；由于每桶贡献的区间长度等于其照片数，**照片多的桶被选中的概率正比于其照片数**——这就是加权随机。
- 选定桶后，`immich_album_page_for_count()` 计算该桶应从第几页取（每页 `page_size` 张，随机选页），保证桶内也能均匀覆盖。
- `count == 0 ? 1 : bucket.count`：空桶给最小权重 1，既避免除零，也防止某些月份完全不被选中。这种防御性编程在嵌入式环境很重要。

### 三、幻灯片状态机——下载完成处理（slideshow_controller.h）

`SlideshowController` 是整个相框的「大脑」。它管理 3 个环形 slot（图像槽），采用**当前显示 + 双预取**策略。当一个 slot 下载完成时，`handle_slot_download_finished()` 决定下一步动作：

```cpp
enum SlideshowAction : uint8_t {
  SLIDESHOW_ACTION_NONE = 0,
  SLIDESHOW_ACTION_DISPLAY_CURRENT = 1,
  SLIDESHOW_ACTION_START_ACTIVE_PAIR = 2,
  SLIDESHOW_ACTION_FETCH_COMPANION = 3,
  SLIDESHOW_ACTION_PREFETCH = 4,
};

static SlideshowAction handle_slot_download_finished(int slot, SlotMeta &meta,
                                                       SlotFlags &flags,
                                                       int &noncritical_count,
                                                       int &download_retries,
                                                       int active_slot,
                                                       bool portrait_pairing_enabled,
                                                       bool &active_slot_displayed,
                                                       DisplayMeta &current_display,
                                                       PortraitState &portrait,
                                                       int &companion_target_slot,
                                                       int portrait_preload_slot,
                                                       std::string &portrait_search_datetime,
                                                       std::string &portrait_primary_asset_id) {
  if (!handle_slot_download_complete(slot, meta, flags, noncritical_count, download_retries))
    return SLIDESHOW_ACTION_NONE;                  // 下载实际未成功，直接返回
  bool is_active = active_slot == slot;
  bool pair = meta.is_portrait && portrait_pairing_enabled;
  if (is_active) {
    copy_slot_to_display(meta, current_display);   // 把元数据拷到当前显示状态
    if (!pair) active_slot_displayed = true;       // 横图：标记已显示
  }
  if (is_active && !pair) {
    return SLIDESHOW_ACTION_DISPLAY_CURRENT;       // 活动横图：立即显示
  }
  if (is_active && pair) {
    if (!(active_slot_displayed && portrait.is_pair)) {
      return SLIDESHOW_ACTION_START_ACTIVE_PAIR;   // 活动竖图：启动配对流程
    }
    return SLIDESHOW_ACTION_NONE;                  // 已在配对中，等待
  }
  if (!is_active && pair) {
    if (companion_target_slot == slot || portrait_preload_slot == slot) {
      return SLIDESHOW_ACTION_NONE;                // 这正是等待的配对图，无需再取
    }
    companion_target_slot = slot;                  // 记录配对目标
    portrait_search_datetime = meta.datetime;       // 用同一天去搜索配对图
    portrait_primary_asset_id = meta.asset_id;
    return SLIDESHOW_ACTION_FETCH_COMPANION;       // 非活动竖图：触发取配对图
  }
  return SLIDESHOW_ACTION_PREFETCH;                // 普通预取完成：可继续预取
}
```

**逐段解析：**

- **状态机入口**：`handle_slot_download_complete()` 先确认下载真的成功（更新重试计数、非关键错误计数），失败则返回 `NONE`，不触发任何显示动作。
- **活动槽判定**：`is_active` 表示刚下载完的正是当前应显示的 slot。横图（`!pair`）直接 `DISPLAY_CURRENT`；竖图则进入配对流程 `START_ACTIVE_PAIR`。
- **竖图配对核心**：当一张竖图下载完成（非活动槽），控制器记录其 `datetime` 和 `asset_id`，返回 `FETCH_COMPANION`，让上层用「同一天」为条件搜索另一张竖图。两张同日竖图并排显示，宽屏利用率大幅提升。
- **幂等保护**：`companion_target_slot == slot || portrait_preload_slot == slot` 防止重复触发配对请求——如果这张图本就是等待中的配对图，直接返回 `NONE`。
- 函数返回的是**动作枚举**而非直接执行，这是一种命令模式：把「决策」和「执行」分离，便于上层在合适时机（如 loop 迭代）执行 HTTP 调用，避免在回调深处发起网络请求导致重入。

### 四、优先级下载队列 FetchQueue（slideshow_controller.h）

3 个 slot 的下载需要排队，但不同任务有不同优先级（当前要显示的 > 预取的）。作者实现了一个**容量固定的最大优先级队列**：

```cpp
class FetchQueue {
 public:
  static constexpr size_t CAPACITY = 6;
  bool enqueue(FetchJobKind kind, int slot, uint8_t priority, uint32_t now_ms) {
    if (this->contains(kind, slot)) return true;   // 去重：同任务已在队列则不重复入队
    if (this->count_ >= CAPACITY) return false;    // 队列满则丢弃（预取本就可丢）
    FetchJob job;
    job.kind = kind;
    job.slot = slot;
    job.priority = priority;
    job.queued_ms = now_ms;
    this->jobs_[this->count_++] = job;
    return true;
  }
  bool pop(FetchJob &out) {
    if (this->count_ == 0) return false;
    size_t best = 0;
    for (size_t i = 1; i < this->count_; i++) {
      if (this->jobs_[i].priority > this->jobs_[best].priority) best = i;  // 找最高优先级
    }
    out = this->jobs_[best];
    for (size_t i = best + 1; i < this->count_; i++) {
      this->jobs_[i - 1] = this->jobs_[i];         // 删除后前移填补
    }
    this->count_--;
    return true;
  }
 private:
  FetchJob jobs_[CAPACITY]{};
  size_t count_ = 0;
};
```

**逐段解析：**

- **静态数组** `jobs_[CAPACITY]`：嵌入式环境避免动态内存，容量 6 足够覆盖 3 slot + 配对 + 预取任务。
- `contains()` 去重：同一个 `(kind, slot)` 任务已在队列就不重复入队，防止预取请求堆积。
- `pop()` 是线性扫描找最大 `priority`，O(n) 但 n≤6，比堆实现更简单且无内存碎片。弹出后用前移填补空位。
- 优先级设计：活动槽配对（`FETCH_JOB_COMPANION`）最高，普通 slot 预取给较低值（如下文 20/10）。

预取入队逻辑清晰体现了优先级差异：

```cpp
static bool enqueue_prefetch_slots(FetchQueue &queue, int active_slot,
                                     const SlotMeta &slot0, const SlotMeta &slot1,
                                     const SlotMeta &slot2, const SlotFlags &flags,
                                     uint32_t now_ms) {
  queue.clear();
  int next1 = (active_slot + 1) % 3;               // 紧邻的下一个 slot
  int next2 = (active_slot + 2) % 3;               // 再下一个
  const SlotMeta &n1 = get_slot_const_(next1, slot0, slot1, slot2);
  const SlotMeta &n2 = get_slot_const_(next2, slot0, slot1, slot2);
  if (!n1.ready && !flags.fetch_in_flight[next1]) {
    queue.enqueue(FETCH_JOB_SLOT, next1, 20, now_ms);  // 优先级 20
  }
  if (!n2.ready && !flags.fetch_in_flight[next2]) {
    queue.enqueue(FETCH_JOB_SLOT, next2, 10, now_ms);  // 优先级 10
  }
  return !queue.empty();
}
```

紧邻下一张（`next1`）优先级 20，更远的（`next2`）优先级 10，确保资源紧张时先取马上要显示的。同时检查 `!flags.fetch_in_flight[next1]` 避免对正在下载的 slot 重复请求。

### 五、远程图像流式下载与格式检测（remote_image.cpp）

`OnlineImage` 是整个项目技术含量最高的部分。它要在 MCU 有限内存下，**边下载边解码**，并自动识别 JPEG/PNG/WebP/BMP 四种格式。核心在 `update()` 和 `loop()` 两个方法。

`update()` 发起请求并根据 HTTP 状态决定策略：

```cpp
void OnlineImage::update() {
  if (this->decoder_ || this->downloader_) {
    ESP_LOGW(TAG, "Cancelling in-progress image download to fetch new URL");
    this->end_connection_();                       // 中断进行中的下载
  }
  std::vector<http_request::Header> headers = {};
  http_request::Header accept_header;
  accept_header.name = "Accept";
  // 根据配置格式设置 Accept，帮助 Immich 选合适缩略图
  switch (this->format_) {
    case ImageFormat::JPEG:  accept_mime_type = "image/jpeg"; break;
    case ImageFormat::PNG:   accept_mime_type = "image/png";  break;
    case ImageFormat::WEBP:  accept_mime_type = "image/webp"; break;
    default:                 accept_mime_type = "image/*";
  }
  accept_header.value = accept_mime_type + ",*/*;q=0.8";
  headers.push_back(accept_header);
  // 注入用户自定义头（如 Immich API Key）
  for (auto &header : this->request_headers_) {
    headers.push_back(http_request::Header{header.first, header.second.value()});
  }
  this->downloader_ = this->parent_->get(this->url_, headers,
      {ETAG_HEADER_NAME, LAST_MODIFIED_HEADER_NAME, CONTENT_TYPE_HEADER_NAME});
  int http_code = this->downloader_->status_code;
  if (http_code == HTTP_CODE_NOT_MODIFIED) {
    // 服务器返回 304：图片没变，跳过下载
    ESP_LOGI(TAG, "Server returned HTTP 304 (Not Modified). Download skipped.");
    this->end_connection_();
    this->download_finished_callback_.call(true);
    return;
  }
  if (http_code != HTTP_CODE_OK) {
    this->end_connection_();
    this->download_error_callback_.call();
    return;
  }
  ImageFormat resolved = this->detect_format_();   // 先用 Content-Type 判断格式
  if (resolved == ImageFormat::AUTO) {
    // Content-Type 缺失或错误：延迟到读到文件头魔数再判断
    this->data_start_ = nullptr;
    this->start_time_ = ::time(nullptr);
    this->last_progress_millis_ = millis();
    this->enable_loop();                           // 进入 loop() 边读边检测
    return;
  }
  this->decoder_ = this->create_decoder_for_format_(resolved);  // 立即创建解码器
  // ...prepare 后进入 loop 流式下载
}
```

**逐段解析：**

- **304 优化**：高频率轮播时，同一张图可能被反复请求。利用 ETag/Last-Modified 条件请求，服务器返回 304 时直接跳过下载，回调 `download_finished_callback_(true)`（true 表示未变化）。注释解释了为何**主动不发送**条件验证器：在高频轮播场景下，反复 304/取消循环会导致显示管线状态竞争。这里只在缓存命中时才享受 304 收益，是权衡后的选择。
- **格式检测两段式**：先用 `Content-Type` 判断（快），失败则进入「魔数检测」模式——读文件头字节比对 JPEG(`FF D8`)、PNG(`89 50 4E 47`)、WebP、BMP 签名。这能应对服务器错误标注 MIME 的情况。
- `enable_loop()` 把工作交给 ESPHome 主循环，避免阻塞。

`loop()` 是真正的流式引擎，交替执行「下载」和「解码」：

```cpp
void OnlineImage::loop() {
  bool made_progress = false;
  // ... AUTO 格式检测分支（前 12 字节）...

  if (this->decoder_->is_finished()) {
    // 解码完成：仅在此时发布 width_/height_，防止半解码图像闪屏
    this->data_start_ = buffer_;
    this->width_ = buffer_width_;
    this->height_ = buffer_height_;
    this->etag_ = this->downloader_->get_response_header(ETAG_HEADER_NAME);
    this->last_modified_ = this->downloader_->get_response_header(LAST_MODIFIED_HEADER_NAME);
    this->end_connection_();
    this->download_finished_callback_.call(false);
    return;
  }
  // 下载阶段：从 HTTP 连接拉数据到环形缓冲
  size_t available = this->download_buffer_.free_capacity();
  if (available) {
    auto len = this->downloader_->read(this->download_buffer_.append(), available);
    if (len > 0) { this->download_buffer_.write(len); made_progress = true; }
  }
  // 解码阶段：有缓冲数据就喂给解码器
  if (this->download_buffer_.unread() > 0) {
    auto fed = this->decoder_->decode(this->download_buffer_.data(), this->download_buffer_.unread());
    if (fed > 0) { this->download_buffer_.read(fed); made_progress = true; }
  }
  if (made_progress) {
    this->last_progress_millis_ = millis();
  } else if ((millis() - this->last_progress_millis_) > DOWNLOAD_STALL_TIMEOUT_MS) {
    this->fail_download_("Download stalled while downloading image",
                         http_request::HTTP_ERROR_CONNECTION_CLOSED);
  }
}
```

**逐段解析：**

- **延迟发布尺寸**：`data_start_`/`width_`/`height_` 只在 `is_finished()` 后才设置。注释点明原因：ESPHome 显示代码把非零尺寸视为可绘制，若中途设置会导致**半解码图像闪屏**。这是对显示管线的深刻理解。
- **下载-解耦交替**：每次 loop 先 `read` 拉一段数据，再 `decode` 喂一段数据。`download_buffer_` 是环形缓冲，`unread()` 返回未消费字节，`fed` 是解码器实际消费的字节数，消费后 `read(fed)` 推进读指针。这种生产者-消费者模式让下载和解码**重叠进行**，而非「下完再解」。
- **停滞超时**：`DOWNLOAD_STALL_TIMEOUT_MS` 内若无进展则判定连接卡死，主动失败。这对无线网络不稳定场景至关重要，避免相框卡在某张图上无限等待。

### 六、RGB565 像素写入与字节序处理（remote_image.cpp）

ESP32-P4 的 Guition 屏以 RGB565 小端序存储像素，而其他目标可能需要大端。解码器统一输出 `Color`，由 `draw_pixel_()` 负责字节序转换：

```cpp
void OnlineImage::draw_pixel_(int x, int y, Color color) {
  if (x < 0 || y < 0 || x >= this->buffer_width_ || y >= this->buffer_height_) {
    ESP_LOGE(TAG, "Tried to paint a pixel (%d,%d) outside the image!", x, y);
    return;
  }
  uint32_t pos = this->get_position_(x, y);
  switch (this->type_) {
    case ImageType::IMAGE_TYPE_RGB565: {
      this->map_chroma_key(color);                 // 色度键透明处理
      uint16_t col565 = display::ColorUtil::color_to_565(color);
      if (this->is_big_endian_) {
        this->buffer_[pos + 0] = static_cast<uint8_t>((col565 >> 8) & 0xFF);  // 高字节在前
        this->buffer_[pos + 1] = static_cast<uint8_t>(col565 & 0xFF);
      } else {
        // 小端序：低字节在前（Guition 屏）
      }
      break;
    }
    case ImageType::IMAGE_TYPE_GRAYSCALE: {
      auto gray = static_cast<uint8_t>(0.2125 * color.r + 0.7154 * color.g + 0.0721 * color.b);
      this->buffer_[pos] = gray;
      break;
    }
    // ... BINARY 分支 ...
  }
}
```

**逐段解析：**

- `color_to_565()` 把 24 位 Color 压缩为 16 位（R5G6B5）。灰度转换用 ITU-R BT.601 系数（0.2125R + 0.7154G + 0.0721B），符合人眼对绿色更敏感的特性。
- 字节序由 `is_big_endian_` 控制，把硬件差异集中在一处，解码器无需关心屏幕接线方式——这是良好的抽象分层。
- `map_chroma_key()` 处理透明色：ESPHome 保留一个近黑色绿色值作为透明键，真实像素若碰巧等于该值会被推开一位，半透明像素则映射到该键值。

### 七、主组件封装（espframe_component.h）

整个相框逻辑挂载在 ESPHome 组件系统下，结构非常简洁：

```cpp
namespace esphome {
namespace espframe {
class EspFrameComponent : public Component {
 public:
  EspFrameSlideshow &slideshow() { return this->slideshow_; }
  const EspFrameSlideshow &slideshow() const { return this->slideshow_; }
 protected:
  EspFrameSlideshow slideshow_{};
};
}  // namespace espframe
}  // namespace esphome
```

`EspFrameComponent` 继承 ESPHome `Component`，持有 `EspFrameSlideshow` 实例。ESPHome 框架会自动调用其 `setup()`/`loop()` 生命周期方法，幻灯片引擎由此接入主循环。YAML 中通过 `espframe:` 域配置，lambda 通过 `id(espframe).slideshow()` 访问运行时状态。

---

## API 使用指南

### Immich API Key 获取

在 Immich Web 界面：**管理 → API 密钥 → 新建**，复制生成的 Key。

### cURL 模拟相框的搜索请求

```bash
# 随机搜索 5 张图片（模拟 build_immich_search_body）
curl -X POST "https://your-immich/api/search/random" \
  -H "x-api-key: YOUR_IMMICH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "size": 5,
    "type": "IMAGE",
    "visibility": "timeline",
    "withExif": true,
    "isFavorite": true
  }'
```

### cURL 获取图片缩略图

```bash
# 通过 asset_id 获取缩略图（相框实际下载的 URL）
curl -H "x-api-key: YOUR_IMMICH_API_KEY" \
  "https://your-immich/api/assets/{ASSET_ID}/thumbnail?size=preview" \
  -o photo.jpg
```

### Python 脚本：统计时间线桶（模拟加权随机逻辑）

```python
import requests

IMMICH_URL = "https://your-immich"
API_KEY = "your-api-key"
headers = {"x-api-key": API_KEY}

# 获取按月分组的照片时间线
resp = requests.post(
    f"{IMMICH_URL}/api/timeline/bucket",
    headers=headers,
    json={"size": "MONTH", "visibility": "timeline", "withExif": True}
)
buckets = resp.json()

# 加权随机选桶（对应 pick_immich_timeline_bucket_from_choices）
import random
total = sum(b["count"] for b in buckets)
pick = random.randint(0, total - 1)
seen = 0
for b in buckets:
    seen += b["count"]
    if pick < seen:
        print(f"选中桶: {b['timeBucket']}，含 {b['count']} 张照片")
        break
```

---

## 编译与部署

### 方式一：浏览器 Web Installer（推荐普通用户）

1. 用 USB-C 数据线连接屏到电脑（需 Chrome/Edge 桌面版）。
2. 打开 [jtenniswood.github.io/espframe/install](https://jtenniswood.github.io/espframe/install)。
3. 点击连接，选择串口，浏览器直接刷写工厂固件。
4. 刷写完成后，屏幕进入 WiFi 配网向导，输入 Immich 地址和 API Key。

### 方式二：本地编译（开发者）

| 配置项 | 值 |
|--------|------|
| ESPHome 版本 | 2026.6.4 |
| 构建方式 | Docker（免装环境） |
| 工厂固件配置 | `builds/guition-esp32-p4-jc8012p4a1.factory.yaml` |
| 运行时配置 | `builds/guition-esp32-p4-jc8012p4a1.yaml` |

```bash
# 编译工厂固件（Docker 方式，无需本地安装 ESPHome）
docker run --rm -v "${PWD}:/config" \
  ghcr.io/esphome/esphome:2026.6.4 \
  compile /config/builds/guition-esp32-p4-jc8012p4a1.factory.yaml

# 编译产物可直接用 esptool 或浏览器刷写
```

### 烧录与首次配置

1. 刷入工厂固件后，设备启动 WiFi 配网 AP。
2. 手机连接该 AP，完成 WiFi 设置。
3. 在设备 Web 页面（`http://<设备IP>`）填入：
   - Immich 服务器地址（如 `https://photos.example.com`）
   - Immich API Key
   - 照片源（全库/收藏/相册/人物/标签/那年今日）
4. 保存后相框开始轮播。可在 Web 页面调整亮度、色调、轮播速度、日期过滤。

---

## 项目亮点与适用场景

### 技术亮点

- **MCU 上的完整相框引擎**：在无操作系统的 ESP32-P4 上实现了照片搜索、加权随机、流式解码、状态机调度，工程完成度高。
- **边下载边解码**：`OnlineImage::loop()` 的生产者-消费者模型让 HTTP 下载与图像解码重叠，显著降低单张照片就绪延迟。
- **魔数格式检测**：不盲信 Content-Type，用文件头签名兜底，鲁棒性强。
- **竖图配对**：针对竖图在宽屏留白的问题，用「同日搜索」配对并排显示，是产品级的体验优化。
- **304 条件请求 + 防重入设计**：高频轮播下既省流量又避免显示管线状态竞争。
- **header-only 工具函数**：Immich API 工具全部 `inline` 在头文件中，可直接被 ESPHome YAML lambda 调用，无需额外编译单元。

### 适用场景

| 场景 | 说明 |
|------|------|
| 家庭私有云相框 | 连接自建 Immich，客厅展示全家照片 |
| 自托管爱好者 | 不信任云相框服务，要求照片不出局域网 |
| ESPHome 进阶学习 | 学习自定义组件、HTTP 流式下载、LVGL 集成 |
| 二次开发参考 | 加权随机算法、优先级队列、状态机模式可复用 |
| Home Assistant 集成 | 作为 ESPHome 设备接入 HA 自动化 |

---

## 总结

Espframe 把「私有云相框」这个通常需要平板或树莓派才能实现的需求，压缩进了一颗 ESP32-P4 MCU。它的价值不仅在于功能完整，更在于源码中处处体现的工程权衡：手工拼 JSON 以避免依赖、加权随机保证照片公平展示、下载解码重叠以降低延迟、304 与防重入以适应高频轮播。对于想深入学习 ESPHome 自定义组件开发和嵌入式图像处理的人来说，这是一个高质量的参考实现。

---

📝 作者：蔡浩宇（jun-chy）  
📅 日期：2026-07-03  
🔗 项目地址：[jtenniswood/espframe](https://github.com/jtenniswood/espframe)
