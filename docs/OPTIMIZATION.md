# Zipic 代码审查与优化方案

> 静态审查，未在 macOS 上编译或运行。标注"待验证"的条目需要在真机上确认。
> 引用格式：`文件:行号`。

## 一、Bug 清单

### P0：会丢数据或产出错误结果

| # | 问题 | 位置 | 说明 / 触发场景 | 建议修复 |
|---|------|------|----------------|----------|
| 1 | **默认设置具有破坏性** | `Models.swift:108-109` | 默认 `saveMode = .overwriteOriginal`，且拖入后自动压缩。默认输出为 WebP，只有源文件扩展名与输出格式匹配（默认下为 WebP）时才原地覆盖；JPEG/PNG/GIF/SVG/HEIC/TIFF/BMP 等默认写入同目录的 .webp 文件。同名目标冲突另见 #3。原地覆盖的备份只在临时目录且没有"还原"入口。 | 默认改为 `.originalFolder`；覆盖模式首次使用时二次确认；提供"还原原图"。 |
| 2 | **输出可能比原图更大却仍被采用** | `CompressionEngine.compressRasterAsset` | 已优化过的 JPEG 重新编码、PNG 经 CoreGraphics 重编码、GIF 重编码都可能变大；覆盖模式下直接替换原文件。UI 用 `max(0, …)` 把负节省隐藏成 "节省 0%"。 | 结果 ≥ 原图且格式未变时，保留原文件（覆盖模式）或不输出，并在备注中说明。 |
| 3 | **覆盖模式、改格式时会静默覆盖无关文件** | `destinationURL` `:634-648` | `photo.png → webp` 时目标名为 `photo.webp`，且覆盖模式跳过了"重名加序号"逻辑；同目录已有 `photo.webp` 会被直接替换。备注还写着"已安全输出为新文件"。 | 覆盖模式下目标与原文件不同时，同样做重名检查。 |
| 4 | **jpegtran 临时文件可能误伤用户文件** | `optimizeJPEGInPlace` `:756-770` | 临时文件放在用户目录，名为 `<name>-opt.jpg`；若用户已有同名文件，会被 `-outfile` 覆盖、随后被 `defer` 删除。另外先 `removeItem(url)` 再 `moveItem`：move 失败时（磁盘满/权限）覆盖模式下的原文件已经被删。 | 临时文件放系统临时目录（UUID 命名）；用 `FileManager.replaceItemAt` 原子替换。 |
| 5 | **EXIF 方向丢失，手机照片可能被转 90°** | `loadRasterImage` `:348` | 不缩放时走 `CGImageSourceCreateImageAtIndex`，不会应用方向信息；而 `render()` 重绘后又丢掉了 EXIF，于是 iPhone 竖拍照片输出横躺。缩放路径使用了 `CreateThumbnailWithTransform`，所以"是否缩放"会导致结果方向不一致。同时 `pixelWidth/Height` 取的是未旋转尺寸，方向 5–8 时宽高被互换。 | 两条路径统一用 `CreateThumbnailAtIndex(... WithTransform: true, MaxPixelSize: 原图长边)`；或读取方向后手动变换，并据此交换宽高。 |
| 6 | **SVG 缩放会改写所有 `width`/`height`（含子元素和 `stroke-width`）** | `replaceSVGAttribute` `:687-694` | 正则 `width\s*=\s*"…"` 全局替换，会命中 `<rect width>`、`stroke-width`、`<image width>` 等。设置了最大宽度后 SVG 内部图形尺寸被破坏。当前测试只断言了 `width="400"` 出现，没发现问题。 | 只解析并改写根 `<svg>` 标签的属性；若有 `viewBox` 就只改根 `width/height`，不动内部元素。 |
| 7 | **动画 GIF 在默认设置下丢失动画** | `compress` `:83-107` | 只有 `(.gif → .gif)` 才走动画路径。默认输出格式为 WebP，GIF 被当作静态图只取第 1 帧；动画 WebP/APNG 同理。界面没有任何提示。 | 动画源 + 非动画目标格式时：保持原格式或在备注/弹窗中明确提示；检测 APNG / 动画 WebP 的帧数。 |
| 8 | **WebP 输出依赖 Homebrew 的 `cwebp`（待验证）** | `encodeWebP` `:430`、`tool(named:)` `:772` | 仅在 `/opt/homebrew/bin` 等路径查找 `cwebp`。没装的机器会走 ImageIO 回退；据我所知 ImageIO 不支持写 WebP，会抛"无法创建导出目标"。而 WebP 恰好是默认输出格式。 | 方案见下文"三、1"：内置 libwebp；至少启动时检测，不可用则禁用该选项并提示。 |

### P1：行为错误或体验缺陷

| # | 问题 | 位置 | 说明 | 建议 |
|---|------|------|------|------|
| 9 | **压缩过程中再拖入新文件，前一批剩余文件永远卡在"等待中"** | `AppState.compress` `:193-257` | 每次 `compress` 都换一个 `compressionRunID`，旧任务下一轮检测到 ID 不同就直接 `return`，旧批次未处理的条目无人接手；正在处理的那张文件已写入磁盘，但状态不会更新。 | 改成单一串行队列：新文件追加入队，不取消现有任务；仅"清空"才取消。 |
| 10 | **无法取消、无总进度** | `MainWindowView.swift:80-83` | 压缩期间"清空列表"被禁用，目标大小模式下大图可能迭代很久，只能强退。 | 加"停止"按钮（`Task` 取消 + 引擎内检查 `Task.isCancelled`）；加整体进度条。 |
| 11 | **`JPG 输出已自动铺白透明背景` 提示永远不出现** | `compressRasterAsset` `:278` | 检查的是 `rendered.alphaInfo`，而 `rendered` 已经被铺白成 `noneSkipLast`，`containsAlpha` 恒为 false。 | 检查 `raster`（渲染前）的 alpha。 |
| 12 | **目标大小模式达不到目标时无提示** | `:282-287` | 缩放到 20% 仍超标，循环退出后照样输出超标文件，无备注。 | 达不到目标时追加备注，或标记状态。 |
| 13 | **目标大小模式极慢** | `encodeJPEG` `:388` + 外层循环 | 外层缩放最多约 15 轮，每轮 JPEG 二分 9 次编码，且每轮重新解码原图。最坏约 130+ 次编码。 | 先在原尺寸做质量二分，只有质量下限仍超标才缩放；解码结果复用；先用估算比例直接跳到合适的缩放值。 |
| 14 | **PNG 基本没被压缩，且"压缩强度"对 PNG 无效** | `encodeImage` `:367` | 只是 CoreGraphics 重编码，不做调色板量化/优化。PNG→PNG 经常变大（见 #2）。 | 接入 `pngquant`/`oxipng`（或 libimagequant）；至少文档说明。 |
| 15 | **GIF 在"压缩强度"模式下根本不压缩** | `compressAnimatedGIF` `:206-211` | 非目标大小模式第一轮就 break，只是重编码，常常更大。 | 引入 gifsicle / 降帧 / 减色；或提示"GIF 仅支持目标大小模式"。 |
| 16 | **GIF 帧解码失败时被跳过，但目标帧数仍按总数创建** | `:187-189` | `CGImageDestinationCreateWithData(..., frameCount, ...)` 与实际添加帧数不一致，Finalize 可能失败或产出损坏文件。 | 先收集成功帧，再用实际帧数创建 destination。 |
| 17 | **WebP 回退质量公式无意义** | `:456` | `targetBytes / 500_000` 与图像尺寸无关，等于固定质量+靠缩小尺寸达标。 | 回退路径也做质量二分。 |
| 18 | **"原图大小"可能被读成压缩后的大小** | `loadPreviewAndMetadata` `:276-289` | 覆盖模式下第 1 张图的压缩任务可能先于元数据读取完成，读到的是已被覆盖的文件。同时压缩完成后 `dimensionsText` 被替换成输出尺寸，原始尺寸丢失。 | 完成时用 `CompressionResult.originalSize`（来自备份，正确）回填；原/新尺寸分开展示。 |
| 19 | **主线程做文件拷贝** | `prepareInputURL` `:259-274`（拷贝在 `:270`） | `AppState` 是 `@MainActor`，`copyItem` 在主线程执行，大图/批量时界面卡顿。 | 备份拷贝挪到非隔离的后台函数。 |
| 20 | **"重新压缩"在非覆盖模式下不断产生新文件** | `recompressAll` + `destinationURL` | 每次重新压缩都会生成 `-compressed-2`、`-3`…，旧输出不会清理。 | 重新压缩时复用/替换上次输出，或提示。 |
| 21 | **输入框体验** | `optionalIntBinding` `:170-181` | 输入任何非数字字符（如 `12a`）会把值设为 `nil` 并清空输入框；`intBinding` 无法清空到空。 | 过滤非数字字符而不是置 nil；用 `TextField(value:format:)` 或格式化器。 |
| 22 | **压缩期间改设置，同一批次前后参数不一致** | `compress` `:223` | 每张图在处理时才读取 `settings`。 | 开始时一次性快照设置。 |
| 23 | **多个失败只弹最后一个** | `AppState.swift:246` | `alertMessage` 被覆盖，信息丢失。 | 汇总到批次结尾一次性展示，或仅在行内显示。 |
| 24 | **不支持拖入文件夹** | `importFiles` `:104` | 目录会被扩展名过滤掉，无任何反应。 | 递归枚举目录内的受支持文件。 |

### P2：SVG、工程与可维护性

| # | 问题 | 说明 |
|---|------|------|
| 25 | **SVG 尺寸解析不严谨**：`svgCanvasSize` 的正则会命中 `stroke-width` 或子元素的 `width`；`width="100%"`、`10mm` 之类被当作 100px、10px；`viewBox` 不支持逗号分隔。 |
| 26 | **SVG "精简"可能改变渲染**：`>\s+<` 会去掉 `<tspan>` 之间的空白，`\s{2,}` 会压缩 `<text>`/`<pre>`/`<style>` 内的空白，注释正则也会误删 CDATA 里的内容。建议使用真正的 XML 解析，或接入 svgo。 |
| 27 | **SVG 在默认输出格式（WebP）下被栅格化**：矢量图变位图，用户通常不期望。建议 SVG 默认保持 SVG。 |
| 28 | **`outputSuffix` 无界面入口**，用户无法修改 `-compressed`。 |
| 29 | **设置持久化脆弱**：`CompressionSettings` 新增字段后，旧版本 JSON 解码失败会静默重置全部设置。应使用 `decodeIfPresent` 或带版本号。 |
| 30 | **临时备份目录靠 `deinit` 清理**：进程退出不会触发 `deinit`，`ZipicSession-*` 目录会遗留；备份在会话期间一直占用磁盘。应在 `applicationWillTerminate` 清理，启动时扫描清理旧目录。 |
| 31 | **元数据（EXIF/ICC/GPS）被统一丢弃**，且无选项。隐私上是好事，但摄影用户可能需要保留版权/色彩配置。建议加"保留元数据"开关；P3 色域图被强制转 sRGB。 |
| 32 | **模块命名遗留**：库名 `TuyaCore` 与产品 `Zipic` 不一致。 |
| 33 | **仓库提交了二进制产物** `dist/Zipic-arm64.zip`，且 `docs/RELEASE.md` 要求每次发布都提交，仓库会持续膨胀；`.gitignore` 里的 `/.omx` 与项目无关。建议改用 GitHub Releases。 |
| 34 | **打包**：仅 ad-hoc 签名，无公证，用户下载后会被 Gatekeeper 拦截；`swift-stdlib-tool` 在 macOS 14+ 上拷贝 Swift 运行库多半没必要；版本号写死 `1.0.0`。 |
| 35 | **测试覆盖很薄**（仅 JPEG 目标大小、SVG）：没有覆盖覆盖模式、重名、方向、GIF、透明 PNG→JPG、输出变大等路径；现有 `ZipicTests` 只依赖 `TuyaCore`；可添加对可执行 target `Zipic` 的依赖，通过 `@testable import Zipic` 测试 `AppState`，无需先拆分模块。 |
| 36 | `inspectImage` 在导入时和压缩时各读一次；`runProcess` 先 `waitUntilExit` 再读 stderr，输出超过管道缓冲会死锁（概率低）；`Package.swift` 要求 swift-tools-version 6.3，限制了贡献者的工具链。 |

## 二、优化方案（按阶段）

### 阶段 1：先止血（数据安全 + 明显错误，建议优先，约 1–2 天）

1. 默认改为"原文件夹 + 后缀"，覆盖模式加确认、加"还原"（#1）。
2. 覆盖模式下目标重名检查（#3）；jpegtran 改用临时目录 + `replaceItemAt`（#4）。
3. 输出不小于原图时保留原文件，并在 UI 中显示"已是最优"（#2）。
4. 修 EXIF 方向（#5）、SVG 根节点属性改写（#6）、JPG 铺白提示（#11）。
5. 队列改单一串行队列，修复卡死条目（#9），加取消按钮（#10）。
6. 补测试：覆盖重名、方向、SVG 内部 `width`/`stroke-width` 不被改、输出变大。

### 阶段 2：核心压缩质量（约 1 周）

1. **内置编码器**：把 libwebp（以及可选的 mozjpeg、oxipng/libimagequant、gifsicle）作为 SwiftPM C target 或打包进 `Resources`，不再依赖 Homebrew 路径（#8、#14、#15）。
2. 目标大小算法重写：先质量二分，再按估算比例缩放，解码结果缓存（#13、#17）；达不到目标时给出提示（#12）。
3. 动画格式：检测动画 WebP/APNG/GIF；动画源遇到不支持动画的目标格式时，保持原格式或明确提示（#7、#16）。
4. SVG 改用 XML 解析（`XMLParser`）或 svgo，修正尺寸解析（#25、#26）。

### 阶段 3：体验与工程化

1. 有限并发（如 `max(1, activeProcessorCount / 2)`），带内存上限，保留"逐张释放"特性；增加总进度条与每行进度。引入并发前，必须串行分配并保留目标路径，或原子预占目标文件，避免不同目录的同名输入在共享输出目录中竞争并覆盖彼此结果（#3）。
2. 支持文件夹拖入、"保留元数据"开关、后缀设置界面、原/新尺寸分开展示（#18、#24、#28、#31）；修正数字输入的过滤与清空行为（#21）。
3. 直接为 `ZipicTests` 添加 `Zipic` 依赖并补齐 `AppState` 单测（#35）；业务逻辑下沉到 core 模块可作为独立架构改进，不是补测试的前置条件。
4. 设置快照与版本化持久化（#22、#29）；退出时清理备份目录（#30）。
5. 发布流程：二进制改走 GitHub Releases；加 Developer ID 签名 + 公证；版本号来自 git tag（#33、#34）。

## 三、几点说明

1. 关于 #8（ImageIO 是否能写 WebP）：这是我的经验判断，没有在真机验证；请在没有 Homebrew 的 Mac 上选择"输出 WebP"测一次。
2. 关于 #5（EXIF 方向）：逻辑上必现，建议用一张 iPhone 竖拍 JPEG（不设置宽高）验证。
3. 以上行号基于当前 `main`（commit `67cd69f`）。
