# 混音舱 · HunyinCang

**Android 音频耦合触感播放器**

混音舱是一款**完全离线**的本地音乐播放器。它把 Android 的 audio-coupled haptic feedback（音频耦合触感）能力落地为可日常使用的播放体验：在支持的设备上，马达振动由音频链路直接驱动，声音与触感天然同步，而不是靠定时器或波形分析去「猜」节拍。应用不申请网络权限，解码、元数据解析与转码全部在本机完成。

> 本项目 README 依据《混音舱 · 开发者技术文档 v1.0》撰写。文档对应的应用版本为 v1.0（versionCode 1）；本仓库随附的 `混音舱_1.2.apk` 为 v1.2（versionCode 14），架构与实现一脉相承。

---

## 核心能力

| 能力 | 实现要点 | 依赖条件 / 备注 |
| --- | --- | --- |
| **音频耦合触感播放** | `HapticGenerator.create(audioSessionId)` 挂到 MediaPlayer 会话，配合 `setHapticChannelsMuted(false)` | API 31+ 且 `isAvailable()` 为真；不支持时自动降级为普通播放。触感实际效果需真机验证 |
| **文件夹分组主页** | 基于 `MediaStore.RELATIVE_PATH / BUCKET_DISPLAY_NAME` 分组，组内按歌曲名排序 | 曲目需位于共享媒体目录并被系统索引 |
| **歌曲名与封面** | 标题缺失时回退文件名；封面优先读文件内嵌图，再回退 MediaStore | 内嵌封面支持 FLAC / OGG / MP3 |
| **模糊搜索** | 多关键词 AND，命中方式为「包含」或「跳字子序列」 | 纯内存过滤，作用于歌曲名、歌手、文件夹名 |
| **四种播放模式** | 顺序 / 列表循环 / 单曲循环 / 随机，队列取自当前可见列表 | 单曲循环用 `seekTo(0)`，不重建 HapticGenerator |
| **内嵌歌词** | FLAC Vorbis Comment、OGG comment、MP3 ID3v2 USLT/SYLT，同名 `.lrc` / `.txt` 兜底 | 纯 Kotlin 字节解析，无第三方库；GBK 自动回退 |
| **播放详情页** | 大图封面、同步高亮歌词、可拖动进度条、上一首/暂停/下一首 | 歌词带时间戳时自动滚动 |
| **深浅色主题** | SharedPreferences 持久化 `dark` 布尔值 | 重启后保持 |
| **离线转码 OGG** | 解码 PCM → JNI 编码为 3 声道 Vorbis，写入 `ANDROID_HAPTIC=1` | SAF 目录授权；保留标题、歌手、歌词、封面 |
| **调试日志** | 播放器状态变更写入最近 40 条日志，设置页可查看 | 用于真机定位触感链路 |

---

## 运行环境与硬约束

| 项目 | 取值 / 约束 |
| --- | --- |
| 最低 / 目标 SDK | minSdk 29（Android 10） / targetSdk 34 / compileSdk 34 |
| 触感能力门槛 | `SDK_INT >= 31` 且 `HapticGenerator.isAvailable()` 为真；低版本先短路，不解析 API 31 方法 |
| 触感策略 | 不做降级补偿：不支持时就是普通播放，不尝试用 Vibrator 模拟 |
| 权限 | `VIBRATE`；读取音频按 SDK 分支 `READ_MEDIA_AUDIO`（33+）/ `READ_EXTERNAL_STORAGE`（≤32） |
| 网络 | **不申请任何网络权限**，不联网、不上传、无广告 |
| 进程模型 | 单进程、单 Activity、无 Service、无前台通知；切到后台不保证继续播放 |
| 架构 | 单 Gradle 模块 `:app`；Kotlin + Compose；native 仅 `libhapticogg.so` |
| 产物 ABI | `arm64-v8a` / `armeabi-v7a` / `x86_64`（均 16KB 页对齐） |

> **关于「需真机验证」**：所有涉及 `HapticGenerator` 实际振动效果、音量行为、设备 HAL 表现的结论，模拟器与静态检查只能验证代码路径，无法验证马达是否真的按预期振动，因此一律标注需真机验证。

---

## 技术栈

- Kotlin 1.9.24 · Jetpack Compose（Material3，BOM 2024.06.00）· JDK 17
- Gradle 8.7 + AGP 8.5.2（Gradle 9.x 与 AGP 8.5.2 不兼容，故固定 8.7）
- `MediaPlayer` / `AudioAttributes` / `AudioManager` / `audiofx.HapticGenerator`
- `MediaStore` / `MediaExtractor` / `MediaCodec` / `DocumentsContract`（SAF）
- JNI + `libogg 1.3.5` + `libvorbis 1.3.7`（编码子集，静态编入 `libhapticogg.so`），NDK r26d clang
- 无第三方标签解析库、图片加载库、播放器框架、网络 SDK；所有字节解析与 DSP 均为手写实现

---

## 系统架构

工程没有引入 MVVM / Repository / Hilt 等框架，而是按「数据来源 → 控制 → 表现」三层组织，全部集中在两个 Kotlin 文件中。链路短、调试日志可覆盖全流程；代价是 UI 与解析逻辑耦合。

```
表现层   MainActivity / HapticScreen   ModalNavigationDrawer（主页/文件夹/转码/设置/软件介绍）
         Scaffold: TopAppBar + 搜索框 + LazyColumn + MiniPlayer
         DetailScreen / AboutScreen / AboutSection
                              ↓ 状态与事件 ↓
控制层   HapticPlayer（MediaPlayer + HapticGenerator）  PlayMode 队列与完成回调
         armVolumeGuard 音量防御 · onLog → logs / tick 刷新 · SharedPreferences
                              ↓ 数据读取 ↓
数据层   MediaStore.Audio.Media → Song / Group · fuzzy / subsequence / Collator 排序
         纯 Kotlin 标签解析（歌词/封面） · artCache / lyricCache
                              ↓ 离线批处理 ↓
转码层   MediaExtractor + MediaCodec → PCM · HapticOgg（JNI）· libhapticogg.so · SAF 写出
```

状态管理没有使用 ViewModel / StateFlow，而是用 Compose 的 `remember { mutableStateOf(...) }`，并把播放器设计成**日志驱动**：`HapticPlayer` 内部任何状态变化都调用 `onLog`，`tick++` 触发重组，让迷你播放条、模式按钮、设置页日志同步刷新。`DisposableEffect` 的 `onDispose` 保证 Activity 销毁时严格释放 HG 与 MediaPlayer。

### 三条主链路

- **播放（实时触感）**：`MediaStore URI → generateAudioSessionId → MediaPlayer.create(attrs, sid) → HapticGenerator.create(session) → start() + 音量防御`
- **转码（离线预置触感）**：`MediaExtractor 选音轨 → MediaCodec 解码 PCM → nativeWrite [L][R][haptic] → cacheDir 临时 OGG → SAF 目标目录`
- **歌词/封面**：IO 线程解析，`artCache` / `lyricCache`（`ConcurrentHashMap`）缓存

### HapticGenerator 八条硬约束

1. 只在 API 31+ 且 `isAvailable()` 为真时使用，低版本先短路版本判断
2. `MediaPlayer.create` 必须用带 `AudioAttributes` 的五参重载，且 `.setHapticChannelsMuted(false)`
3. 必须绑定正确的 `audioSessionId`（`mp.audioSessionId` 作为 `HapticGenerator.create()` 入参）
4. `VIBRATE` 权限必须声明；读音频按 SDK 33 分支申请
5. 切歌或停止前必须 `setEnabled(false)` → `release()` → `MediaPlayer.release()`
6. 播放过程中不重复 `create()`：单曲循环用 `seekTo(0)`，暂停/拖动进度也不重建
7. Android 12 音量异常需防御（HG 启用后媒体流可能被拉到最大音量）
8. 不做降级补偿：不支持设备就是普通播放；所有触感结论标注需真机验证

---

## 离线转码：三声道 ANDROID_HAPTIC OGG

实时把音频转成可震动版本不可行——HapticGenerator 本身就是实时 effect，无公开 API 把自定义声道写回播放链路，且 Vorbis 有损分块会破坏播放连续性。因此落地为**离线批处理导出**：

- 容器/编码：OGG + Vorbis VBR（quality 0.42）
- 声道布局：`[0]=L`、`[1]=R`、`[2]=haptic`
- 触感标记：Vorbis comment `ANDROID_HAPTIC=1`（完整 KEY=VALUE 写入）
- 透传元数据：`TITLE`、`ARTIST`、`LYRICS`、`METADATA_BLOCK_PICTURE`、`ENCODER=HunyinCang`
- 触感声道 DSP：L/R 下混 → 两级一阶高通（250 Hz）→ 混回低频（0.12×mono）→ 慢包络 AGC（gain ∈ [0.4, 2.5]）→ 软削波
- 写出：先写 `cacheDir/export_{id}.ogg`，再经 SAF `createDocument` 复制到用户选定目录
- native 库显式使用 `-Wl,-z,max-page-size=16384` 满足 Android 15+ 的 16KB 页对齐要求

> 用本应用播放导出的 3 声道 OGG 时，需先在设置里关闭「触感播放」，否则实时 HapticGenerator 生成的触感会与文件里预置的触感声道叠加。

---

## 工程结构（精简）

```
HunyinCang/
├── settings.gradle                 rootProject.name = "HunyinCang"; include(":app")
├── build.gradle                    AGP 8.5.2 / Kotlin 1.9.24
├── keystore/debug.keystore         release 与 debug 共用，保证签名一致
└── app/src/main/
    ├── AndroidManifest.xml         权限 / label="混音舱" / launcher icon
    ├── java/com/hunyincang/
    │   ├── HapticPlayer.kt         播放控制、HG 生命周期、队列、音量防御
    │   └── MainActivity.kt         UI、MediaStore、标签解析、封面、转码、JNI 声明
    ├── jniLibs/{abi}/libhapticogg.so
    └── res/mipmap-*/               ic_launcher 图标
```

关键 Gradle 决策：`namespace` 与 `applicationId` 同为 `com.hunyincang`；`minifyEnabled false`（R8 会重命名 JNI 方法导致 `UnsatisfiedLinkError`）；debug 与 release 均 `debuggable false`（部分 ROM 对可调试应用弹兼容性提示）；debug / release 共用同一 keystore，可直接覆盖安装。

---

## 构建与验证

```bash
# Gradle 8.7（独立 --gradle-user-home 绕开全局 init.gradle）、JDK 17、SDK 34、NDK r26d
./gradlew --gradle-user-home <isolated-home> assembleRelease
zipalign -c -p 4 app/build/outputs/apk/release/app-release.apk
apksigner verify --verbose app/build/outputs/apk/release/app-release.apk
```

产物验证记录（v1.0）：`assembleRelease` 成功、`apksigner verify` 通过、`com.hunyincang` / versionCode 1 / versionName 1.0 / label「混音舱」、`debuggable` 计数为 0；三个 ABI 的 `libhapticogg.so` 均 `LOAD` 段 `p_align = 0x4000`；JNI 符号 `Java_com_hunyincang_HapticOgg_*` 三个 ABI 各 3 个，旧包符号残留 0。

---

## 已知限制与演进方向

| 限制 / 风险 | 当前缓解 |
| --- | --- |
| 设备 HAL 差异导致「有声无感」 | 设置页明示设备状态、调试日志可查；不做补偿，需真机验证 |
| 无后台 Service / 音频焦点管理 | 单 Activity 轻量实现，切后台不保证继续播放 |
| 外部 `.lrc` 读取依赖 MediaStore 索引 | 三级回退（内嵌 → MediaStore.Files → File） |
| 3 声道 OGG 兼容性 | 导出为独立文件，不影响原曲 |
| 实时触感与预置触感叠加 | 介绍页提示先关闭触感播放开关 |
| UI 与数据层耦合在单文件 | 建议拆分 Repository / ViewModel |

**建议演进顺序**：① 拆分 Repository / ViewModel 并补单元测试；② 引入 MediaSession / 前台 Service / AudioFocus；③ 按 ABI 分包减小体积；④ 触感参数做成设置项配合真机 A/B 调参；⑤ 转码支持批量队列与失败重试。

---

## 安装与使用

1. 下载 `混音舱_1.2.apk` 安装到 Android 10+ 设备。
2. 首次启动授予音乐/存储权限；主页按文件夹列出歌曲。
3. 触感播放需要 Android 12+（API 31+）且设备支持 audio-coupled haptic HAL 且 `HapticGenerator.isAvailable()` 为真；不支持时自动降级为普通播放。
4. 建议把媒体音量调到 2–3 格，选鼓点明显的曲子，握持手机感受振动；同时可在设置页打开「显示调试日志」对照排查。

| 文件 | 说明 |
| --- | --- |
| `混音舱_1.2.apk` | 安装包，19,552,985 字节，SHA-256 `af37a66b7e40a034f67d6fc1a0593f4a8994969bd987ca01bf316d6d87aea1da` |
| `HunyinCang-Developer-Docs-v1.0.pdf` | 开发者技术文档 v1.0，31 页，含架构、核心实现、转码、构建验证与真机验证清单 |

---

*文档版本 v1.0（2026-11-07）· 应用标识 com.hunyincang · 所有触感行为请以真机验证结论为准。*
