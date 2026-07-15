# Kazumi 离线超分辨率 (Offline Super-Resolution) — 交接文档

> 交接日期: 2026-07-15
> 制作者: OpenClaw Agent (qql2)
> 后续接手工具: Claude Code (Codex CLI)
> Fork 仓库: `qql2/kazumi-offline-sr`
> 上游仓库: `Predidit/Kazumi` (main 分支, v2.2.0, commit count: 1,711+)

---

## 一、项目背景

### 1.1 目标

为 Kazumi 增加**离线超分辨率**功能，解决低性能设备上实时 Anime4K 超分导致的 GPU 高负载、风扇噪音问题。

**核心思路**：在播放之前，先利用下载好的视频文件，通过更高质量的 ML 模型逐帧处理，生成超分后的缓存文件，然后播放时直接使用超分后的本地文件。这样播放时 GPU 只负责普通解码，零额外压力。

### 1.2 工作流程（目标架构）

```
用户操作：番剧详情 → 选择"离线超分" → 选择集数
                               ↓
                    [后台管线 — 完全异步]
                    ① 下载视频（复用现有下载系统）
                    ② 合并 TS 片段（如源为 HLS）
                    ③ 解码帧 → ML 推理逐帧超分 → 编码
                    ④ 保存超分后文件
                    ⑤ 更新缓存索引（episode → 超分文件路径）
                               ↓
                    [播放 — 零改动]
                    用户打开该集 → 检测到本地超分文件 → 直接播放
```

---

## 二、项目现状分析

### 2.1 已有能力（直接可复用）

| 功能模块 | 现状 | 对你的价值 |
|---------|------|-----------|
| **番剧下载** | ✅ 已上线，支持 HLS 和直链下载 | TS 片段和直链文件已存于本地 |
| **离线播放** | ✅ 已支持从本地路径播放 | 播放器端无需改动 |
| **缓存管理** | ✅ Hive + DownloadRepository | 可复用 cache-first 模式 |
| **弹幕缓存** | ✅ cache-first 策略 | 超分后播放弹幕正常 |
| **投屏/硬解** | ✅ DLNA + Vulkan/OpenGL | 超分后体验不变 |
| **跨平台** | ✅ Android/iOS/Windows/macOS/Linux | 推理引擎需适配各平台 |

### 2.2 缺失能力（需要你实现）

| 模块 | 说明 |
|------|------|
| **ML 推理引擎** | Flutter/Dart 侧无原生推理支持，需通过 `dart:ffi` 调用 C++ 推理库 |
| **视频帧处理管线** | 解码 → 帧缓存 → 逐帧推理 → 编码输出的完整管道 |
| **超分状态管理** | 新增 DownloadEpisode.status 枚举值（`upscaling`, `upscaled` 等） |
| **UI 入口** | 下载页面增加"超分下载"按钮，或在播放器设置中增加"离线超分后播放" |
| **存储策略** | 超分后文件的管理（自动清理、空间预留） |
| **合并 TS 工具** | 部分视频源存储为 TS 片段，需要 ffmpeg concat 支持 |

---

## 三、代码架构详解

### 3.1 技术栈

```
Flutter 3.44.6 (Dart 3.3.4+)
├── 状态管理: flutter_modular + mobx
├── 网络请求: dio
├── 持久化存储: hive_ce
├── 媒体播放: media-kit (Flutter 包装) → libmpv (底层)
├── 实时超分: Anime4K GLSL shaders (assets/shaders/)
└── 元数据: Bangumi API + 弹弹play Danmaku API
```

### 3.2 关键目录结构

```
kazumi-offline-sr/
├── lib/
│   ├── main.dart                  # 应用入口
│   ├── core_module.dart           # 核心模块注册
│   │
│   ├── modules/download/
│   │   └── download_module.dart   # 下载数据模型 (Hive 实体)
│   │     ├── DownloadRecord       # 番剧下载记录
│   │     ├── DownloadEpisode      # 单集下载详情
│   │     └── DownloadStatus       # 状态枚举
│   │
│   ├── services/download/
│   │   ├── download_manager.dart  # 下载管理器（核心逻辑）
│   │   └── background_download_service.dart
│   │
│   ├── repositories/
│   │   └── download_repository.dart # 下载数据持久化层
│   │
│   ├── pages/download/
│   │   ├── download_page.dart       # 下载页面 UI
│   │   ├── download_controller.dart # 下载逻辑控制器
│   │   ├── download_widgets.dart
│   │   └── download_episode_sheet.dart
│   │
│   ├── pages/player/               # 播放器
│   │   ├── controller/             # 播放控制
│   │   └── ...                     # 弹幕、UI 等
│   │
│   ├── services/shaders/
│   │   └── shader_asset_service.dart # Anime4K shader 加载
│   │
│   ├── utils/
│   │   ├── m3u8_parser.dart         # M3U8 解析
│   │   ├── m3u8_ad_filter.dart      # 广告过滤
│   │   └── file_system.dart        # 文件系统工具
│   │
│   └── services/storage/
│       └── storage.dart             # 全局存储 (Hive boxes)
│
├── assets/shaders/                  # Anime4K GLSL shader 文件
│
├── android/                         # Android 原生层
├── ios/                             # iOS 原生层
├── macos/                           # macOS 原生层
├── windows/                         # Windows 原生层
└── linux/                           # Linux 原生层
```

### 3.3 下载数据模型（⚠️ 重点：你需要扩展此模型）

```dart
// lib/modules/download/download_module.dart —— 核心数据模型

@HiveType(typeId: 7)
class DownloadRecord {
  int bangumiId;           // Bangumi.tv ID
  String bangumiName;      // 番剧名称
  String bangumiCover;     // 封面
  String pluginName;       // 视频源插件名
  Map<int, DownloadEpisode> episodes;  // 集数映射
  DateTime createdAt;
  String get key => '${pluginName}_$bangumiId';  // 唯一键
}

@HiveType(typeId: 8)
class DownloadEpisode {
  int episodeNumber;        // 集数编号
  String episodeName;       // 集名称
  int road;                 // 线路编号
  int status;               // 0=pending, 1=resolving, 2=downloading,
                             // 3=completed, 4=failed, 5=paused
  double progressPercent;   // 下载进度 0.0~1.0
  int totalSegments;        // TS 片段总数
  int downloadedSegments;   // 已下载片段数

  String localM3u8Path;     // ⭐ 播放时使用的本地路径
                             //   HLS: 本地 playlist.m3u8 路径
                             //   直链: 本地 video.mp4 路径
  String downloadDirectory; // 下载目录
  String networkM3u8Url;    // 原始网络 M3U8 URL
  DateTime? completedAt;

  int totalBytes;           // 总字节数
  String episodePageUrl;
  String danmakuData;       // 缓存的弹幕 (JSON)
  int danDanBangumiID;      // DanDanPlay 番剧 ID
}
```

### 3.4 下载管理器核心流程

**HLS 下载流程** (`download_manager.dart`):

```
enqueue(request)
  → _runEpisodeDownload()
    → 创建 ${downloadBase}/${bangumiId}_${pluginName}/${episodeNumber}/ 目录
    → 解析 M3U8（支持 master/media/nested）
    → 解析密钥（AES-128 decrypt keys）
    → 并发下载 TS 片段（maxParallelSegments = 3）
    → 生成本地 playlist.m3u8（指向本地 TS 片段）
    → 设置 episode.localM3u8Path = 本地 m3u8 路径
    → 设置 episode.status = completed
    → 设置 episode.localM3u8Path
```

**直链下载流程**:

```
enqueue(request) [URL 检测为非 M3U8]
  → _runDirectFileDownload()
    → 直接流式下载 → 保存为 video.mp4
    → episode.localM3u8Path = video.mp4 路径
```

⭐ **关键**：播放器通过 `DownloadManager.getLocalVideoPath(episode)` 获取本地文件路径，任何可被 mpv 播放的文件路径都可以。这意味着你只需**输出一个本地文件路径**，播放器就能无缝播放。

### 3.5 播放器集成

播放器通过 `media-kit`（Flutter 包）调用 `libmpv`。核心配置：

```
vo=libmpv           # 渲染输出
hwdec=no            # 默认关闭硬解（可在设置开启）
glsl-shaders=...    # Anime4K GLSL shaders（实时超分路径）
demuxer-cache-dir=... # 播放缓存目录
```

播放本地下载文件时，mpv 直接打开 `episode.localM3u8Path`（m3u8 或 mp4），**不再使用 Anime4K shader**（因为普通播放不需要超分）。

---

## 四、技术选型建议

### 4.1 ML 推理引擎对比

| 选项 | 协议 | 移动端支持 | Dart FFI 绑定 | 模型生态 |
|------|------|-----------|---------------|---------|
| **ONNX Runtime** | MIT | ✅ Android/iOS | ✅ 有社区 flutter_onnx | 丰富（Real-ESRGAN 等） |
| **NCNN** | BSD-3 | ✅ 优秀 | ⚠️ 需自写 | 丰富（Waifu2x, Real-ESRGAN） |
| **TFLite** | Apache-2.0 | ✅ 优秀 | ✅ 官方 flutter | 较少动漫专用模型 |

**推荐 NCNN 或 ONNX Runtime**：
- NCNN：腾讯开源，对移动端优化极好，很多动漫超分项目用它
- ONNX Runtime：模型生态最好，跨平台支持最完善

### 4.2 推荐 ML 模型（动漫专用）

| 模型 | 质量 | 速度 | 协议 | 备注 |
|------|------|------|------|------|
| **Real-ESRGAN (anime)** | ⭐⭐⭐⭐⭐ | 慢（>100ms/帧） | BSD-3 | 质量最佳，适合离线 |
| **Waifu2x (CUnet)** | ⭐⭐⭐⭐ | 中等（~50ms/帧） | MIT | 经典动漫超分 |
| **FSRCNNX** | ⭐⭐⭐ | 快（~6ms/帧） | MIT | 轻量但质量一般 |
| **Anime4K (ML 版)** | ⭐⭐⭐⭐ | 快（~10ms/帧） | MIT | 和已有 shader 同源 |

**MVP 建议**：先用 Waifu2x 或 Real-ESRGAN 做概念验证，因为模型易获取、社区成熟。

### 4.3 平台层集成策略

```
Dart (Flutter)                 原生层 (C++/Kotlin/Swift)
┌─────────────────┐           ┌──────────────────────┐
│                  │ dart:ffi  │                      │
│ download_manager │──────────→│  ML Inference Engine  │
│                  │           │  (ONNX / NCNN)       │
│ upscale_service  │←──────────│                      │
│                  │           │  解码 → 推理 → 编码   │
└─────────────────┘           └──────────────────────┘
```

具体来说：
- **跨平台**：Android/iOS/Linux/macOS → C++ 共享代码 + `dart:ffi`
- **Windows**：DLL + `dart:ffi`
- Flutter 项目已有的 `android/`、`ios/`、`windows/`、`macos/`、`linux/` 各目录下已有原生代码框架

---

## 五、MVP 实施路线

### Phase 1：环境搭建与验证（预计 1-2 天）

```bash
# 1. Fork 已在 qql2/kazumi-offline-sr
# 2. 确保开发环境
cd kazumi-offline-sr
flutter doctor    # 确认 Flutter 3.44.6 + Dart SDK
flutter pub get

# 3. 本地运行验证
flutter run -d macos   # 或 windows/android

# 4. 手动验证下载 + 播放管线
#  - 打开应用 → 搜索一部番剧 → 下载一集
#  - 找到下载目录验证文件
#  - 确认离线播放正常
```

### Phase 2：核心技术预研（预计 3-5 天）

1. **ffmpeg 集成**：在原生层添加 ffmpeg 支持，实现 TS 合并 → 完整视频
2. **推理引擎绑定**：选择一个推理库，写 Dart FFI 绑定（MVP 用 ONNX Runtime）
3. **逐帧处理管线**：解码帧 → 送入模型 → 接收输出 → 编码

```python
# 先在 Python 中验证效果（工具链确认）
# 下载一集 → Python 脚本处理 → 对比效果
ffmpeg -i downloaded_video.mp4 -vf "format=yuv420p" -f rawvideo pipe: | \
  python3 run_esrgan.py | \
  ffmpeg -f rawvideo -pix_fmt yuv420p -s 1920x1080 -i pipe: output.mp4
```

### Phase 3：MVP 集成（预计 5-10 天）

1. **扩展数据模型**：在 `DownloadEpisode` 增加超分相关字段
2. **创建离线超分服务**：`lib/services/upscale/` 目录，包含状态管理
3. **UI 入口**：在下载页面增加"超分下载"选项
4. **播放器集成**：检测本地超分文件 → 优先播放

### Phase 4：发布与优化（持续）

1. 平台兼容性测试
2. 进度显示、暂停/取消
3. 缓存清理策略
4. 可选模型选择

---

## 六、现有 Issue 与社区信号

### 6.1 相关 Issue

| Issue | 状态 | 说明 |
|-------|------|------|
| [#1843](https://github.com/Predidit/Kazumi/issues/1843) | closed (not_planned) | "添加更多超分模型"，被关闭。表明作者目前对扩展超分方向谨慎 |
| [#316](https://github.com/Predidit/Kazumi/issues/316) | closed (completed) | "实现番剧下载"，内有作者对 HLS 转码性能的讨论，非常有参考价值 |
| [#1534](https://github.com/Predidit/Kazumi/issues/1534) | open | 播放器 bug，含 libmpv 日志信息 |

### 6.2 社区交流渠道

- Telegram: https://t.me/kazumi_app
- GitHub Issues: https://github.com/Predidit/Kazumi/issues
- 规则仓库: https://github.com/Predidit/KazumiRules

### 6.3 作者风格分析

从 Predidit 的对话历史看（Issues #316 等）：
- **务实派**：重视用户体验和性能，拒绝不可行的方案
- **透明度高**：会解释技术决策背后的原因
- **对核心质量要求高**：不愿意为了进度而妥协体验
- **沟通最好用中文**

---

## 七、注意事项与风险

### 7.1 GIFM 协议合规

- 上游 Kazumi 是 **GPL-3.0**，你的 fork 继承此协议
- 选择 ML 模型时需注意其许可证（Real-ESRGAN: BSD-3 ✅, Waifu2x: MIT ✅）
- 推理引擎需 GPL 兼容（ONNX Runtime: MIT ✅, NCNN: BSD-3 ✅）

### 7.2 TS 文件格式

**这是下载模块本身都没完全解决的问题**：
- HLS 源 = 数十个 TS 片段 + m3u8 索引
- 简单 concat 可能在某些播放器上失败（非标准 HLS 流）
- 完整转码（libx264）在骁龙 8Gen3 上也要 ~5 分钟/20分钟番剧

**建议 MVP 方案**：
1. **优先针对直链视频源**（下载为完整 mp4 的源），跳过 TS 合并问题
2. 对于 HLS 源：先 concat 成 TS → 再用 ffmpeg 转码为 mp4（或对每个 TS 片段独立处理帧）
3. mpv 本身可以播放 `concat:file1.ts\|file2.ts` 格式，可以不合并直接用

### 7.3 存储空间

- 一集 24min 1080p 番剧 ≈ 200-400MB
- 超分后 1080p→4K 可能膨胀到 1-2GB
- 建议增加"超分文件自动清理"策略（如 7 天未访问自动删除）
- 借鉴已有的"低内存模式"设计思路

### 7.4 移动端特殊考虑

- Android/iOS 后台处理时间长，系统可能杀进程
- 建议实现"后台下载+超分"协同：
  - 下载完成后自动开始超分
  - 超分过程中保持 WakeLock
  - 支持暂停/恢复

### 7.5 上游合并策略

- MVP 阶段建议在自己 fork 上独立开发，不必急于向上游提 PR
- 等 MVP 验证可行后，先开 Discussion 与 Predidit 讨论
- 合并时可以考虑标注为"实验性功能"，降低维护承诺

---

## 八、Claude Code 启动指引

```bash
# 1. 进入项目目录
cd ~/.openclaw/workspace/kazumi-offline-sr

# 2. 启动 Claude Code（如果作为独立 CLI）
# codex 或 claude -p "读取 OFFLINE_SR_HANDOVER.md 了解项目背景"

# 3. 首次要做的几件事
#    a. 确认 Flutter 环境可用
flutter doctor

#    b. 拉取依赖
flutter pub get

#    c. 在本地跑起来验证
flutter run -d macos  # 或 windows/android

#    d. 测试下载功能：选一部番剧下载一集，确认文件落盘路径
#    e. 找到下载目录，确认文件结构
```

## 九、参考资料

- **上游仓库**: https://github.com/Predidit/Kazumi
- **Fork 仓库**: https://github.com/qql2/kazumi-offline-sr
- **Anime4K**: https://github.com/bloc97/Anime4K
- **media-kit**: https://github.com/media-kit/media-kit
- **ONNX Runtime**: https://github.com/microsoft/onnxruntime
- **NCNN**: https://github.com/Tencent/ncnn
- **Real-ESRGAN**: https://github.com/xinntao/Real-ESRGAN
- **Waifu2x**: https://github.com/nagadomi/waifu2x
- **DeepWiki (Kazumi)**: https://deepwiki.com/Predidit/Kazumi
- **Kazumi 官网**: https://kazumi.app
