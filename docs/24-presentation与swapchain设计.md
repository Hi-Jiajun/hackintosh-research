# presentation / swapchain 设计：从离屏附件到"可读回的呈现目标"（2026-09-14）

> 本文是图形/显示路径第三段（**presentation / swapchain**）的启动设计。`docs/16` 把这条序列
> 钉成依赖序（纹理 → sampler → 计算读纹理 → render pipeline → render pass 与附件 →
> **presentation / swapchain** → heaps / ICB），`docs/23` 完成了 render pipeline 与离屏附件
> 这一段（Step 1–3c 已落地，v13 进入 CI，提交 `8430446`；撰写过程中又落了只改 README 的
> `1529f90`）。本文只定义**呈现轨道的第一段**。
>
> **边界**：本文只新增本文件，不写实现、不改任何代码仓、不 push。不改
> `docs/11-Metal-provider-contract-v0.md`（长期 dirty）、不改其它 docs。所有"现状"结论都在
> 撰写时实地读过代码，行号可用 §8 的 `rg -n` 复现；凡未在代码、既有文档或本次实测中确认的行为，
> 一律标"待确认"，不臆造 API 名称。
>
> **路径约定**：`crates/**`、`conformance/**`、`examples/**` 相对
> `/home/hiliang/hackintosh/metal-api-emulator`（分支 `shared-provider-objects`，撰写时 HEAD
> `1529f90`；代码行号在 `8430446` 上核对，两者只差 README）；`docs/NN-*.md` 相对
> `/home/hiliang/hackintosh/research`（撰写时 HEAD `62f9cdd`）；
> `repos/fork-reims-vgpu/**` 与 `tools/*.sh` 相对 `/home/hiliang/hackintosh`（`tools/` 只在顶层
> 工作区：`metal-api-emulator/tools/` 不存在）。
>
> **相关文档**：`docs/09` §13.3 第 5 条（图形/显示路径未完成）、`docs/16`（图形路径规划：
> 纹理优先）、`docs/23`（离屏渲染第一段）、`docs/13`（回收与错误传播）、`docs/21`（队列优先级与
> family）、`docs/14`/`docs/15`（range hazard 与共享分配）、emulator 仓
> `conformance/RENDER-CAPTURE.md`、`docs/SHARED-PROVIDERS.md`、`README.md` 的 "Not implemented"
> 列表。
>
> **五路径标签**（`conformance/compare.py:21-26`，命令入口 `:692-702`）：`native-metal`（Swift
> oracle）/ `vulkan`（direct trace）/ `native-metal-provider`（Rust native trace）/
> `vulkan-objects` / `native-metal-provider-objects`。`objects --async` 是同一条对象 API 轨的
> 异步捕获，**不是第六个 backend**（`tools/lavapipe-smoke.sh:41`）。

## 1. 问题与目标

### 1.1 为什么 presentation 是渲染之后的下一个里程碑

1. **依赖序上它就是下一个。** `docs/16:11-16` 的序列把 render pass 与附件排在 presentation
   之前，理由是"swapchain 的每一帧就是渲染到可 present 的附件"。离屏附件已经能端到端跑通
   （`docs/23` §1.2、`conformance/RENDER-CAPTURE.md` §6），现在要回答的是"这些字节除了被读回，
   还能不能进入呈现语义"。
2. **它验证的是渲染轨道没有覆盖的三件事。** 离屏渲染证明的是"光栅化 + 附件 store 的字节正确"；
   呈现引入：**目标是谁的**（谁拥有那张 image）、**谁来同步**（present 前必须完成什么）、
   **谁在等谁**（acquire / release 的次序）。这三件事在离屏路径里全被"一次性、自己建、同步等"
   掩盖掉了：`execute_offscreen_render`（`crates/metal-api-vulkan/src/render.rs:433`）每次都新建
   image / framebuffer / pipeline，然后 `queue_submit` + `wait_for_fences`
   （`render.rs:944` 的 `submit_and_wait`），没有"归还"这一步。
3. **它是"帧"的入口。** 有呈现目标才有"多帧"（acquire → 渲染 → present → 下一帧）、resize、
   suboptimal、丢帧这些概念；它们又是 heaps/ICB 之前唯一还需要外部状态的接口。
4. **边界风险必须先被钉死。** 真实呈现要接宿主窗口/显示系统，而那是 display 轨道在管的事
   （仓库根 `HANDOFF-DISPLAY-FIX.md`、`HANDOFF-DISPLAY-ROUND2.md`）。本文最核心的裁决（§1.2 末）
   就是：**第一段不接真实窗口**，先把"呈现语义"在 provider 契约里立起来。

### 1.2 三处可得性先论证（Windows / WSL / Apple）

任务的约束是"先论证该目标在 Windows/WSL/Apple 三处的可得性，再决定是否改用'呈现到内存可读的
交换链等价物'"。逐处结论：

| 环境 | 真实呈现目标（surface + swapchain + present） | 依据 |
|---|---|---|
| Windows（产品宿主，非 WSL） | **可得，但不在 provider 内**：reims 已有 host 窗口与 resident 呈现轨道（`repos/fork-reims-vgpu/crates/reims-vgpu/src/host_window/present.rs`；`backend/window/mod.rs:100-145` 的 `WindowPresentOutcome::Presented{suboptimal,...}` 与 `WindowDecline::{PresenterLost,Refused}`），生产配置 `REIMS_VGPU_WINDOW=on` | 代码存在；`docs/16` 已把真实显示划给 display 轨道 |
| WSL（本机开发，WSLg） | **实测可得**：instance 含 `VK_KHR_surface`(rev 25)、`VK_KHR_xcb_surface`、`VK_KHR_xlib_surface`、`VK_KHR_wayland_surface`、`VK_KHR_display`、`VK_EXT_headless_surface`；dzn 与 llvmpipe 两块设备都暴露 `VK_KHR_swapchain`(rev 70)，可呈现 surface 类型 `[xcb, xlib]` 与 `[wayland]`，格式 `B8G8R8A8_UNORM`/`B8G8R8A8_SRGB`，present mode 4 种（IMMEDIATE/MAILBOX/FIFO/FIFO_RELAXED），`minImageCount=3`、`maxImageCount=0`；`vkcube --c 3` 在两个 ICD 上都**退出码 0**，去掉 `DISPLAY` 的对照组**退出码 1** | §7.1 与 §8.6 的原始输出 |
| Apple GPU | **待确认，且大概率不成立**：oracle 当前是离屏 `MTLTexture`（`conformance/RENDER-CAPTURE.md` §3），`CAMetalLayer` 的 drawable 需要 layer 挂到 view/display；`RENDER-CAPTURE.md` §6 明确列出"渲染路径从未在 Apple 硬件上跑过" | 文档记录，本机无实测手段 |

两条比"可得性"更硬的门禁事实：

- **CI 没有显示服务器。** `tools/lavapipe-smoke.sh:14` 固定 `VK_ICD_FILENAMES=.../lvp_icd.json`，
   整条 smoke 在无 X/Wayland 的环境里跑；`.github/workflows/ci.yml` 的 macOS job 也只跑离屏
   capture。**任何"必须真开一个窗口"的断言都不可能成为 CI 判据。**
- **契约层本来就没有宿主窗口的对端。** `metal-api-core` 是中立契约（`docs/09` §13.3 第 1 条）：
   它的资源模型是 `AllocationId`/`ViewId`/lease，`RenderAttachment`（`provider.rs:1263`）之所以
   是"引用身份 + 重述形状"而不是嵌入 `TextureView`，正是因为契约里不能出现宿主句柄。加一个
   `HWND`/`NSView` 字段会把中立契约变成宿主集成层。

**裁决**：第一段的呈现目标 = **"呈现到内存可读的交换链等价物"**。真实窗口与显示输出留在
display 轨道，不进 provider 契约。§3.6 给出这个等价物的确切口径。

### 1.3 第一个可验证里程碑（一句话）

**一次 present：把一个已渲染的 2×2 颜色附件交给 provider 拥有的"呈现目标"，经
acquire → present 之后该目标在宿主侧可读回，读回的 4 个 texel 与 fixture 期望逐字节一致，
且这次 present 的完成被记录成可断言的计数。**

与 `docs/23` §1.2 的差别：那一条的出口是"附件的字节"，这一条的出口是"经过 acquire/present
生命周期之后的目标字节 + 一次 present 的完成记录"。

### 1.4 里程碑的验收判据（可判定，不含主观项）

- **尺寸与格式**：2×2，与 `docs/23` 的附件同尺寸；格式从 `Rgba8Unorm` 起步（为什么不是
  swapchain 偏好的 `B8G8R8A8_*`，见 §7.3 与 §7.6）。
- **防伪**：呈现目标由 provider 用哨兵预置（沿用 v13 的 `fefefefe` 先例，见
  `conformance/suite-v13.json` 的 `clear_hex`），因此"present 没发生"、"present 了但没写"与
  "写对了"三者可区分。
- **出口**：宿主可见字节、texel 级（`docs/23` §3.5 的口径）；不看窗口像素，不引入图像相似度。
- **计数**：present 次数、acquire 次数、目标 allocation 的 `copy_out` 三项都进报告（§5.3）。
- **明确不进第一段**：真实 surface/窗口、resize 与 recreate、多缓冲（imageCount > 1）、
  垂直同步与 present mode 选择、丢帧与 suboptimal、image acquire 超时、fence 超时策略。

## 2. 现状核对

以下每条都能用 §8 的命令或同形 `rg -n` 复现，行号是**撰写时的行号**（`docs/23` §8.4 引用的
`lib.rs:57` 现在已是 `:72`，说明行号会随提交漂移，引用前必须重跑）。

### 2.1 `crates/metal-api-vulkan/src/lib.rs`：队列族选择与 device 创建

- **队列预算由两个常量约束**：`MAX_QUEUES_PER_FAMILY = 4`（`:67`）、`MAX_DEVICE_QUEUES = 8`
  （`:72`）。后者是 `lib.rs:659` 的 `[0_usize; MAX_DEVICE_QUEUES]` 数组长度，也是真机判据里
  `queues=8` 的来源。
- **family 计划只有两项**：`family_plans` 起手是"主 family"（`select_physical_device` 选中的
  那个，`:473`），再追加**一个 compute-only family**（`:474-497`），其准入条件是"有队列 +
  含 COMPUTE + **不含 GRAPHICS**"（`:477-482`）。**family 计划里没有任何 "present 支持" 的
  概念。**
- **device 创建只带两组队列**：`DeviceQueueCreateInfo` 由 `family_plans` 逐项生成（`:501-505`），
  `queue_create_infos(&queue_infos)`（`:531`）→ `create_device`（`:536`）。创建后立刻把队列
  展平成与索引对齐的两张表（`:543-550`：`queues` 与 `queue_families`），这是
  `select_graphics_queue` 与 `create_command_pool` 的唯一查询面。
- **设备选择不要求 GRAPHICS，更不要求 present**：`select_physical_device`（`:888`）的准入条件
  是 Vulkan 1.3 `maintenance4`、`shaderInt8`、`shaderInt64` 与"至少一个含 COMPUTE 的 family"
  （`:925`）；打分时给含 GRAPHICS 的 family 加分（`:936`）但**不强制**。
- **零 present 代码**：`crates/` 与 `examples/` 里没有 `swapchain` / `VK_KHR_surface` /
  `create_surface` / `queue_present` 的任何出现（§8.5，`rg` 退出码 1）。设备侧也**没有**
  surface 支持查询，device extensions 只按需启用 `VK_EXT_external_memory_host`。

### 2.2 `crates/metal-api-vulkan/src/render.rs`：渲染执行如何选图形队列

- **图形队列是"找出来的"，不是"计划出来的"**：`select_graphics_queue`（`:505-523`）遍历
  `context.queue_families`，对每个 family 重查 `get_physical_device_queue_family_properties`，
  取第一个含 `GRAPHICS` 的**已创建**队列；一个都没有就
  `capability_refusal("render_graphics_queue_unavailable")`（`:520`）。这与 `docs/23` §7.3 的
  判断一致：设备只**可能**有图形族，缺失时是 typed 拒绝而不是静默回退。
- **整套离屏资源的所有权形状**：`execute_offscreen_render`（`:433`）先按能力位预检
  （`admit_color_attachment` `:411` → `format_features` `:371` →
  `format_supports_color_attachment` `:396`），tiling 固定 `OPTIMAL`（`:442`），
  然后 `OffscreenObjects::new`（`:549`）串起 `create_attachment`（`:574`）→
  `create_render_pass`（`:614`）→ `create_framebuffer`（`:655`）→ `create_pipeline`（`:672`）→
  `create_readback`（`:763`）→ `create_command_pool`（`:829`）→ `record`（`:855`）→
  `submit_and_wait`（`:944`）。
- **呈现轨道要接的正是这条链的尾部**。"目标归属"（谁建 image）、"归还"（present 之后目标是
  否可再被 acquire）、"同步"（fence 等待点）都要在这里扩展，而不是另起一条执行路径。
- **`execute_render_pass`（`:266`）是契约到执行的适配层**：它先 `validate_against` 管线契约、
  再查已登记片元 stage 是否匹配附件格式、再逐条拒绝 `LoadOp::Load`/`DontCare` 与
  `StoreOp::DontCare`（`attachment_load_op_unsupported` 等 slug）。presentation 的第一版应当
  沿用这个"先在适配层 typed-refuse，再碰任何 Vulkan 对象"的形状。

### 2.3 `crates/metal-api-core/src/provider.rs`：渲染能力位与 admission

- **能力位已经有四个渲染字段**：`supports_render_passes`（`:3647`）、`max_color_attachments`
  （`:3651`）、`max_attachment_dimension`（`:3654`）、`supported_color_formats`（`:3659`）；
  `declares_render_support()`（`:3666`）是"是否有任何一位离开默认值"的判据，线格式用它决定
  走新 tag 还是旧字节。
- **admission 的渲染门在最前面**：`admit()` 先 `trace.validate()`，紧接着
  `admit_render_passes(trace)`（`:3921`），**早于** pass 数、dispatch 类型、完成策略与资源预约。
  该函数先数 render pass（`render_passes()`，`provider.rs:2895`），为 0 直接返回（compute-only
  流零开销），否则按 `supports_render_passes` / `max_color_attachments` /
  `supported_color_formats` / `max_attachment_dimension` 逐项 typed-refuse（`:3927` 起
  `render_passes_unsupported`）。
- **Vulkan 侧已经声明渲染支持**：`capabilities_from_limits`
  （`crates/metal-api-vulkan/src/provider.rs:30` 起）设 `supports_render_passes: true`（`:58`）、
  `max_color_attachments = 1`（`:59`）、`max_attachment_dimension = MAX_ATTACHMENT_DIMENSION
  = [2, 2]`（`:60`、`:25`）、`supported_color_formats = AttachmentFormat::ADMITTED`（`:61`）。
- **native 侧还关闭着**：`crates/metal-api-native/src/native.rs:181` 是
  `supports_render_passes: false`，`native.rs:173` 留着被注释的翻转条件。呈现必须在渲染之
  后，所以这一位是 presentation 在 macOS 侧的**前置条件**。

### 2.4 `crates/metal-api-core/src/provider.rs`：core 的渲染类型

- `AttachmentFormat`（`:1115`）是**闭集四值**（`Rgba8Unorm`/`Bgra8Unorm`/`R32Float`/`R32Uint`），
  线码与既有纹理格式码相同；`ADMITTED` 只有前三个，`R32Uint` 可表达但被第一增量拒绝。
  **没有 sRGB 变体**——`docs/23` §7.4 的"完全不碰 sRGB"已经写进类型。
- `LoadOp`（`:1215`：`Clear(ClearColor)`/`Load`/`DontCare`）、`StoreOp`（`:1229`：
  `Store`/`DontCare`）、`MAX_COLOR_ATTACHMENTS = 1`（`:1244`）。
- `RenderAttachment`（`:1263`）字段是 `view_id`/`allocation_id`/`format`/`width`/`height`/
  `load`/`store`，即"引用身份 + 重述形状"，宿主侧落地方式完全留给 provider。
- `RenderPassDescriptor`（`:1329`）字段是 `pipeline`/`color_attachments`/`viewport`/`vertices`，
  校验只放行 `(0,0,w,h)` 的 viewport 与 3 个顶点（全屏三角）。**没有 target、没有 acquire、
  没有 present 字段。**
- `VertexLayout`（`:1428`）、`RenderPipelineContract`（`:1451`）与 `TracePass`（`:1539`）：
  `TracePass` 是"计算 / 渲染"的判别联合，`as_compute`/`as_render` 分别取分支，
  `ComputeTrace::render_passes()`（`:2895`）与 `attachments()`（`:2905`）按序铺平。呈现的 pass
  若要进入 trace，就是在这个联合上加第三个臂（或作为渲染 pass 的一个可选尾部动作，§3.5）。

### 2.5 `conformance/`：比较器与 suite 组织（presentation 会不会引入第六路径）

- **五条 backend 是写死的字典**：`ALLOCATION_OBSERVATIONS`（`compare.py:21-26`）把五个名字映射
  到两种观测口径——`native-metal` 是 `gpu-buffer-readback`，其余四条是
  `host-writeback-landing`；CLI 侧就是 `--native`/`--vulkan`/`--metal-provider`/
  `--vulkan-objects`/`--metal-objects`（`:692-702`）。**rail 的粒度是"谁执行这套 trace"，
  不是"哪条执行路径的执行形状"。**
- **渲染用例已经走通了"不新增 rail 也能表达新观测面"这条路**：`_render_plan`（`:361-505`）从
  suite 的 `render_cases` 段建出附件期望，`capture_rails`（`:465`）声明**哪几条 rail 欠这个
  case**：被点名的 rail 漏报要拒绝，没被点名的 rail 多报也要拒绝
  （`conformance/RENDER-CAPTURE.md` §4；`conformance/test_suite_v13.py:220` 是这条规则的测试）。
  附件字节通过**既有** `writebacks`/`allocations` 形状报告，不新开观测通道。
- **count 契约是派生的**：`copy_in = 触及的 distinct allocation 数`、`copy_out = 被写过的
  allocation 数`（`:653-663`）；渲染用例的 `copy_out` 就是那次 `vkCmdCopyImageToBuffer`
  （`conformance/RENDER-CAPTURE.md` §3）。v11/v12 强制上报（`:600-602`），v13 渲染用例只做类型
  校验后 `continue`（`:595-599`）。
- **工具链顺序纪律**：`tools/lavapipe-smoke.sh:25-30` 从 `conformance/suite*.json` 自动发现
  套件，`:35-43` 对每个套件跑 direct / objects / objects-async 三条 Vulkan 捕获。**新 suite
  必须等到 capture 与比较器都支持之后再提交**（`docs/18` 的原始阻断点）。

**结论（回答任务里的"会不会引入第六路径"）**：**不引入。** presentation 不是"另一条实现
rail"，而是同一批 rail 上的一种新 pass / 新目标。判据：`native-metal` 与 `vulkan` 是**执行者**
不同（Swift Metal vs Vulkan），`-objects` 是**接线方式**不同（对象 API vs 直连 trace），
present 两者都要有；一个只会算不能呈现的设备，用**能力位 + `capture_rails` marker**表达即可，
这与 v13 对"两个对象 API 轨没有 render encoder"的处理是同一手法
（`conformance/RENDER-CAPTURE.md` §4）。只有当某个 rail 的呈现语义**无法用同一批断言表达**
时，才需要新 backend 名——第一段不进入那个情形。

### 2.6 宿主侧既有的呈现通道（reims fork）与 provider 的边界

- reims 已经有**窗口/呈现的抽象层**：`crates/reims-vgpu/src/backend/window/mod.rs:100-145` 定义
  `WindowResident`、`WindowPresentOutcome::{Busy,Presented{direct,width,height,buffers,suboptimal}}`
  与 `WindowDecline::{PresenterLost,Refused}`，注释里明确"Metal 的 layer 与 Vulkan 的 swapchain
  都有一个 `buffers` 计数"；`crates/reims-vgpu/src/backend/vulkan/engine/window_present.rs` 是
  MoltenVK swapchain 的 present 引擎；`crates/reims-vgpu/src/host_window/present.rs` 是宿主窗口。
- **它是 display 轨道，不是 provider 契约**：`docs/16:11-16` 把 presentation 放在图形路径序列
  里，但把"真实显示输出"划给显示轨道；本文只做前者，且做成"可读回的等价物"，因此**不需要**
  去动 reims。反过来，reims 已有的 `buffers`/`suboptimal`/`PresenterLost` 词汇是很好的对照：
  第一段**刻意不建模**它们（§3.4），等真正接窗口时再对齐。

### 2.7 现状小结（一句话）

**渲染已经能"写出字节"，但整套执行形状是"自己建、同步等、没有归还"**；presentation 要在
这个形状上补出"目标归属 + 一次提交后的归还/完成记录"两件事，而观测面、五路径、能力位门禁与
加法式线格式都已经有可以照抄的先例。

## 3. 契约设计

### 3.1 最小字段集

第一版只允许**一个呈现目标、一次 present、单缓冲**。字段级清单（新类型，命名按现有风格）：

| 字段 | 类型/取值 | 说明 |
|---|---|---|
| `target` | `PresentTarget` | 见下表；呈现目标的身份与形状 |
| `source` | `ViewId` | 要与 `RenderPassDescriptor.color_attachments[0]` 的同一视图对齐（同一 lease/range 语义） |
| `mode` | `PresentMode::{Fifo}` | 第一版**只放行 `Fifo`**：它是唯一"处处可用"的模式，也是唯一不需要与宿主协商的语义 |
| `acquire` | `AcquirePolicy::Blocking` | 第一版不支持超时（`docs/13` 的 deadline 语义在这里还没有对象可挂） |

`PresentTarget`：

| 字段 | 类型/取值 | 说明 |
|---|---|---|
| `allocation_id` / `view_id` | `AllocationId` / `ViewId` | 与 buffer/texture/attachment 同一命名空间 |
| `format` | `AttachmentFormat` | 第一版必须与来源附件相同（不匹配在 admission 阶段 typed-refuse） |
| `width` / `height` | `u64` | 与 `RenderAttachment::expected_bytes` 同口径（紧排 texel 数） |
| `image_count` | `u32` | 第一版固定 1；"多缓冲"用"字段存在但只接受 1"表达，而不是"字段不存在" |
| `initial` | `InitialState::{Sentinel([u8;4]), Undefined}` | 哨兵让"present 没发生"可证伪（沿用 v13 口径） |

### 3.2 格式与 extent 的来源

- **格式**：**从来源附件继承**，不做"目标格式独立可选"。理由：比较器是字节 parity，两个格式
  一旦不同（哪怕都是 8-bit unorm），通道序与编码就会变成系统性偏差；而且真正的 swapchain 格式
  由 surface 决定（本机实测只有 `B8G8R8A8_UNORM`/`B8G8R8A8_SRGB`，§7.1），第一段既然不接
  surface，就不该假装有选择权。
- **extent**：与来源附件相同（`(0,0,w,h)` 全覆盖）。resize 是 §3.4 的不做项，所以 extent 没有
  第二个来源。
- **谁分配**：目标 allocation 由**调用方按既有 lease 语义预约**，provider 只负责把它变成呈现
  目标（与 `RenderAttachment` "引用身份 + 重述形状"完全同构）。这样 hazard/写回合并规则不需要
  新机制（`docs/14`）。

### 3.3 同步语义

第一版把同步写成**三条可断言的次序约束**，全部落在既有提交/等待生命周期里：

1. **present 前必须完成渲染。** 目标的写入者（render pass）必须在同一提交内先于 present 完成；
   实现上就是同一条命令缓冲里的
   `vkCmdPipelineBarrier(COLOR_ATTACHMENT_OUTPUT → 呈现/传输可见性)` 语义，而不是跨提交的挂起
   状态。
2. **present 后目标可被读回。** 这是本段唯一的出口断言：`wait` 返回后，目标的字节必须已经落在
   宿主可见处（沿用 `docs/23` §3.5 的 `vkCmdCopyImageToBuffer` 路径）。
3. **完成点必须被计数。** present 的完成记一次计数（"present 次数"），acquire 记一次
   （"acquire 次数"），两者都要出现在 provider 报告里，能在 `compare.py` 里与
   `copy_in`/`copy_out` 并列校验。**"present 静默失败"必须表现为计数不符，而不是表现为字节
   不符**（字节可能是上一次的残留）。

**待确认**：`Fifo` 在本段的语义到底是"排队等一个可用目标"（真 swapchain 行为）还是"立即完成
一次目标轮转"（等价物行为）。两种解释对 `acquire` 策略与丢帧语义不同；第一段建议按后者实现并把
差别写进 §9 清单。

### 3.4 明确不做（第一版之外，带触发条件）

| 不做项 | 理由 | 什么时候做 |
|---|---|---|
| 真实 `VkSurfaceKHR` / 窗口 | 契约层没有宿主句柄，CI 没有显示服务器（§1.2） | display 轨道需要 provider 侧对端时 |
| resize / `VK_ERROR_OUT_OF_DATE_KHR` 式重建 | 需要"目标寿命"概念，第一版目标是一次性的 | 多帧循环进来之后 |
| 多缓冲 / imageCount > 1 / acquire 轮转 | 需要"哪张 image 在途"的状态机，与 `docs/13` 的在途资源回收强耦合 | 单缓冲跑通且 `docs/13` 的回收语义定稿后 |
| 垂直同步与 present mode 选择 | 每一种都要跨驱动 parity 证据（本机 4 种 mode 齐全，但 Apple 侧语义不同，§7.5） | 有真实窗口与宿主对端之后 |
| 丢帧 / suboptimal / acquire 超时 | 与 CI 的确定性判据冲突 | 与真实 surface 一起 |
| sRGB 目标与色彩空间匹配 | `docs/23` §7.4 已裁决不碰；surface 偏好的是 sRGB 格式（§7.1），两者冲突 | 有色彩正确性的实际需求时 |

### 3.5 类型草图（字段级，不是可编译代码）

```text
enum PresentMode { Fifo }
enum AcquirePolicy { Blocking }
enum InitialState { Sentinel([u8; 4]), Undefined }

struct PresentTarget {
    allocation_id: AllocationId, view_id: ViewId,
    format: AttachmentFormat, width: u64, height: u64,
    image_count: u32,        // 第一版固定 1
    initial: InitialState,
}

// 形状一：present 作为 render pass 的尾部动作（推荐）
struct RenderPassDescriptor {
    pipeline: PipelineId,
    color_attachments: Vec<RenderAttachment>,
    viewport: [u32; 4],
    vertices: u32,
    present: Option<PresentDescriptor>,   // 新增；None = 现状（纯离屏）
}

struct PresentDescriptor {
    target: PresentTarget,
    source: ViewId,          // 必须是 color_attachments 里的那张
    mode: PresentMode,
    acquire: AcquirePolicy,
}

// 形状二：present 作为独立 pass（trace 联合第三个臂）
enum TracePass { Compute(ComputePass), Render(RenderPassDescriptor), Present(PresentDescriptor) }
```

**推荐形状一**，理由是它把 §3.3 的第 1 条（present 前渲染必须完成）变成**结构上不可能违背**：
present 描述符挂在渲染 pass 上，就不存在"present 一个没渲染的目标"这种 trace。形状二在
"呈现别人的结果"（例如呈现上一个 pass 的附件）时更通用，但会立刻要求一条跨 pass 的资源寿命
规则，属于第一段之后。**这条选择是 §9 待确认第 1 条。**

### 3.6 "呈现到内存可读的交换链等价物"的确切口径

把"交换链"拆成四件可独立建模的事，第一段只做前两件与最后一件，中间那件是真 surface 专属：

| 交换链的组成 | 第一段怎么表达 | 不做的部分 |
|---|---|---|
| 一组可呈现的 image | 目标 allocation/view，`image_count = 1` | 多张 image 的轮转与在途状态 |
| acquire（取一张来写） | 一次"取得目标所有权"的计数 | 超时、BUSY 重试、丢帧 |
| present（交出去） | 一次"目标终态 = 可呈现等价态"的转换 + 计数 | 真正的 `vkQueuePresentKHR` 与合成器队列 |
| 呈现完成后的可见性 | 目标字节在 `wait` 后可从宿主读回 | 屏幕上的像素（属于 display 轨道） |

一句话：**第一段的"呈现"是"目标所有权的一次往返 + 目标内容的宿主可见性"，而不是"屏幕上出现
了一帧"。** 这样它既能被 CI 判据覆盖，又不假装解决了窗口/合成器问题。

## 4. 线格式与能力位

### 4.1 `MCC1` 需要新增什么

沿用 `crates/metal-api-ipc/src/command_codec.rs` 的 `MCC1` 帧与**加法式 tag 先例**
（`SET_QUEUE_PRIORITIES_REQUEST = 0x0e`、`SUBMIT_RENDER_REQUEST = 0x0f`、
`RENDER_CAPABILITIES_RESPONSE = 0x0a`；见 `:52-69`、`:83-84` 的注释与 `:167`/`:223-227` 的
分支）：

- **若选形状一（present 挂在 render pass 上）**：**不需要新 tag**。`SUBMIT_RENDER_REQUEST`
  （`:67`）的 tagged pass 布局里已经有 `PASS_KIND_RENDER`（`:89`）的载荷；present 描述符作为
  该载荷的一个尾部可选段追加，用**长度前缀或 presence 位**区分有无。旧解码器遇到的仍然是
  `0x0f`，它读 render 载荷时会因为多出的尾段而报错——所以**必须**显式约定：旧解码器对
  "render 载荷长度超出旧上限"的行为是 `CodecError`，不是静默截断。这在增量里是新增的一处显式
  拒绝分支，需要单测。
- **若选形状二（独立 present pass）**：新增 `SUBMIT_PRESENT_REQUEST`（建议 `0x10`，**待确认**）
  与 `PASS_KIND_PRESENT`（建议 `0x02`，**待确认**）；旧解码器对 `0x10` 回
  `UnknownCommandTag`，与 `0x0e`/`0x0f` 的既有政策完全同形。
- **不新增**：不改 `COMMAND_FRAME_MAGIC`、不改 completion/lease/borrowed 传输、不改 descriptor
  通道；`PROVIDER_SCHEMA_VERSION` 保持 2（`crates/metal-api-core/src/provider.rs:32`）。

### 4.2 能力位如何声明

与 `AliasMode`/`StorageMode`/渲染四位的既有风格一致：**显式声明、默认关闭、旧 provider 行为
不变**。建议增补（命名待定，风格对照 `provider.rs:3647-3659`）：

| 建议字段 | 第一版取值 | 含义 |
|---|---|---|
| `supports_presentation` | 默认 `false` | 这个快照能不能执行 present |
| `max_present_targets` | 1 | 一个 trace 里最多几个呈现目标 |
| `supported_present_modes` | `[Fifo]` | 只放行 FIFO（§3.1） |
| `max_present_image_count` | 1 | 多缓冲上限（§3.4） |

拒绝路径分两层，与渲染完全同形：core 的 admission typed-refuse（新 slug 建议
`presentation_unsupported` / `present_target_limit` / `present_mode_unsupported`，**待确认**），
provider 在"目标归属 + 同步"那一步再拒一次。`declares_render_support()`（`:3666`）要扩成
"任何一位离开默认值"，否则"声明了 present 但不声明 render"的快照会走旧字节。

### 4.3 旧 trace 解码兼容策略

- **旧字节不变**：compute-only 流仍走 `SUBMIT_REQUEST`（`0x03`）；渲染流仍走
  `SUBMIT_RENDER_REQUEST`（`0x0f`）；只有**真的带 present** 的载荷才改变字节形状（形状一）或
  走新 tag（形状二）。
- **回归判据**：`command::tests::compute_only_submit_keeps_its_pre_render_bytes` 这类"硬编码
  历史字节帧"的测试必须继续保持通过，并且要为 present 增补同形的一条（抓一次改动前的帧字节，
  钉住它不变）。v1–v13 全套件在 `tools/lavapipe-smoke.sh` 下保持绿。
- **不要做**：不要用"升 schema_version"来表达这次变化（`docs/23` §4.3 已用实测结论否掉了升版本
  路线）；不要让 present 与 `air_encoding` 的 raw/wrapped 维度互相放大。

## 5. 五路径比较方式

### 5.1 Swift oracle（`native-metal`）需要什么

真实 Metal 的呈现面是 `CAMetalLayer` + `nextDrawable` + `presentDrawable:`；但 oracle 现在完全
没有 layer（`conformance/RENDER-CAPTURE.md` §3：它渲染到离屏 `MTLTexture`，用 `getBytes`
回读）。第一段的最小集合：

- 一个**`MTLTexture`（`usage` 含 `renderTarget`）充当呈现目标**——这与 `docs/23` §5.1 已经在用
  的用法相同，只是目标变成"经过一次 present 生命周期"的那个；
- 如果坚持要 layer：需要 `CAMetalLayer` 挂到某个 view 或 display 上，第一段**不建议**
  （§7.5，会给 oracle 引入窗口系统依赖与 CI 不可运行性）；
- 出口仍是 `getBytes` 的字节；报告里增加 present/acquire 计数——但 Swift oracle **目前不报
  count**（`compare.py` 的 `provider_backend` 判定在 `:535`），所以计数断言只对 provider 后端
  生效。

### 5.2 Vulkan 三条轨（`vulkan` / `vulkan-objects` / `vulkan-objects-async`）

- **执行侧**：目标是 provider 自有的 optimal tiling image；present 的等价实现是"目标内容 →
  宿主可见字节"的既有 `vkCmdCopyImageToBuffer` 路径（`render.rs:763` 的 `create_readback` 与
  `render.rs:855` 的 `record`），加上一次布局转换到"可呈现等价终态"与一次计数记录。
- **真正要新增的对象只有"目标所有权"这一层**：`OffscreenObjects`（`render.rs:530`）目前每次
  全新创建并全部销毁；呈现目标要么**由调用方持有**（跨提交存活），要么在 provider 内**有
  寿命**。第一版建议前者（目标走 lease，与 `docs/14` 的整资源在途口径一致），这样 `docs/13`
  的回收不需要改。
- **对象 API 轨缺 encoder**：`crates/metal-api-core/src/provider_api.rs:699` 只有
  `compute_command_encoder`；`conformance/RENDER-CAPTURE.md` §4 已经写明"两个对象 API 轨没有
  render command encoder"。present 显然要排在 render encoder 之后。
- **捕获工具**：`examples/metal-smoke/src/bin/provider-capture.rs` 已有三条执行形状（直连、
  `--api objects`、`--api objects --async`，见 `tools/lavapipe-smoke.sh:37-42`）。present 只是
  在同一个二进制里给这三条形状各加一次尾部动作，不新增轨。

### 5.3 观测通道与 count 口径

- **不新开观测通道**：呈现目标的字节走既有的 `writebacks`/`allocations` 形状
  （`conformance/RENDER-CAPTURE.md` §3；`compare.py:361-505` 的 `_render_plan` 就是现成的
  扩展点）。新增的是 **present 段**：`acquire`/`present` 两个计数。
- **建议口径**（**待确认**，需与 v13 的派生规则对齐）：
  - 目标 allocation 是 distinct allocation：`copy_out` 计 1（它被写、也被读回）；
  - `acquire` 计 1、`present` 计 1；丢帧与重试不计（第一版没有）；
  - `copy_in` 视 `InitialState` 而定：`Sentinel` 需要预置字节（1），`Undefined` 是 0——这与
    `docs/23` §5.3 对 `LoadOp` 的处理同构。
- **必须由 marker 声明**：present 用例沿用 `capture_rails`（`compare.py:465`）声明哪几条 rail
  欠它；第一段大概率只有 `vulkan` 一条（与 v13 相同），Apple 侧留给 §5.5 的替代证据。

### 5.4 哪些断言先不要求

- 真实呈现的像素（窗口里看到的东西）、present 时序、VSync 与帧率；
- 丢帧 / suboptimal / `VK_ERROR_OUT_OF_DATE_KHR` 类重建；
- 跨队列或跨提交的 semaphore 语义（第一版是同一提交内的一条链）；
- imageCount > 1 的轮转与"在途目标"的回收；
- sRGB 编码与色彩空间匹配（`docs/23` §7.4 的裁决继续有效）；
- 多目标 / 多附件 / resize。

### 5.5 天然不可比较的 rail 与替代证据形式

- **Apple 侧最可能天然不可比较**：只要呈现目标需要 layer/drawable，`native-metal` 与两条
  `native-metal-provider*` 就无法在 CI 的 Apple Paravirtual 上产出同样的字节。届时的处理与 v13
  完全同构：**`capture_rails` marker 不点名这三条 rail**，改用 `conformance/RENDER-CAPTURE.md`
  §5 的 `--render-selftest` 式"一设备检查"作为替代证据（一个可独立运行、退出码非零即失败的
  JSON 输出，归档到 `evidence/`）。
- **如果连等价物都无法在 Apple 上表达**（例如 `MTLTexture` 的 `storageMode` 组合不允许读回），
  替代证据降级为"对象创建 + 生命周期"级的断言，并在报告里**显式写"该 rail 不参与呈现比对"**，
  而不是让一条 rail 静默缺席。
- **不得**用"图像相似度"或"看起来对"充当替代证据（`docs/23` §1.3 已经定过这条纪律）。

## 6. 实施步骤（每步独立可验证）

**Step 1 — core 纯类型与校验。** 新增 `PresentTarget`（§3.5）/`PresentMode`/`AcquirePolicy`/
`InitialState` 值类型及其校验（格式与附件不符、`image_count != 1`、`mode != Fifo`、哨兵长度与
格式不符、`source` 不在附件列表里）+ 单测。
*验收*：新增单测通过；全仓行为零变化（v1–v13 捕获与测试计数不变）。先例：`docs/14` Step 1、
`docs/16` §4.2 第 1 步、`docs/23` Step 1 都是"纯类型先行、行为零变化"。

**Step 2 — trace 与线格式。** 按 §4.1 选定形状（推荐形状一），扩展 `MCC1` 的 tagged 载荷或新增
tag；能力位加 present 四位（§4.2）；IPC 往返 + "历史字节帧不变"回归。
*验收*：Lavapipe 全套件（v1–v13，含 v8 raw/wrapped 编码）不变；新增 present 往返用例通过；
截断/超长/未知 kind 仍 typed-refuse；`compute_only_submit_keeps_its_pre_render_bytes` 类测试绿。

**Step 3 — Vulkan 轨的无 surface 呈现目标。** 目标 allocation 变为"调用方持有、provider 引用"；
记录 acquire/present 计数；目标终态 = 可读回；`OffscreenObjects` 的所有权形状相应扩展到"目标
不随 pass 销毁"。
*验收*：**本地 Lavapipe** 上 2×2 用例的目标字节 = 期望字节，哨兵不复现；
`provider-capture --suite` 报告能过 `compare.py --check`；目标在 `wait` 之后仍可读（跨提交存活）。

**Step 4 — 观测通道与 count 口径定稿。** `compare.py` 增加 present 段（`acquire`/`present` 与
目标 allocation 的 `copy_out`），新增 `conformance/test_suite_v14.py` 钉住规则：目标字节不能由
buffer writeback 冒充、present 计数不符要拒、未被 marker 点名的 rail 不得上报。
*验收*：纯 Python 测试绿（不依赖 GPU）；合成报告的正反例都覆盖；v1–v13 的 plan 不受影响。

**Step 5 — 对象 API 与第三条 Vulkan 形状。** core 侧 render encoder 之后再补 present 动作
（`provider_api.rs:699` 是现状锚点）；`provider-capture.rs` 的 objects / objects-async 两条形状
接上 present。
*验收*：三条 Vulkan 轨在同一条用例上字节与计数一致；对象 API 的 commit/wait/readback 生命周期
（含 cancel 与 deadline）沿用现有回归。

**Step 6 — 本地真实 surface 探针（非 CI 判据）。** 在 WSL 上做一次 `VkSurfaceKHR` +
`VkSwapchainKHR` + `vkQueuePresentKHR` 的真机探针（形态参照本机 `vkcube`/`vulkaninfo`，§8.6），
回答"真 surface 与等价物差在哪"（格式必须是 `B8G8R8A8_*`、`minImageCount=3`、suboptimal 的
存在感）。产出**只进 `evidence/`**，不进门禁、不改 `tools/lavapipe-smoke.sh`。
*验收*：探针输出归档（驱动名、格式、present mode、imageCount、逐字节结果）；结论写回 §7.1/§7.3。
**RTX 5060 真机**：同一探针在 dzn 上重跑一次（本机 dzn 已有 swapchain 支持，§7.1）。

**Step 7 — native 对称（含前置条件）。** `crates/metal-api-native/src/render.rs` 已有
`MTLRenderPipelineState` 与 `MTLTextureUsage::RenderTarget` 的构造（`:465-498`）；present 等价物
按 §5.1 加到 oracle 与 provider 两侧，能力位按 §4.2 加。
*验收*：**前置条件**是 native 的 `supports_render_passes`（`native.rs:181`）先翻转，而它的翻转
条件是 `conformance/RENDER-CAPTURE.md` §5 的 `--render-selftest` 在 Apple GPU 上通过。
**Apple GPU 验收**：`native-metal` / `native-metal-provider` / `native-metal-provider-objects`
三轨先内部一致，再谈五路径。

**Step 8 — conformance v14 与五路径。** 新增 `suite-v14.json`（2×2 附件 + present），
`capture_rails` 按 §5.5 定；`tools/lavapipe-smoke.sh` 的自动发现**严格后置**（工具先支持，
suite 后提交）；`conformance/README.md` 与 `RENDER-CAPTURE.md` 增补 present 一节。
*验收*：Lavapipe 全套件绿（`LAVAPIPE_SMOKE_OK`）；**RTX 5060 真机**三轨捕获 + `--check` 全 PASS；
五路径 `compare-captures` 在能报 present 的 rail 上一致；证据归档
`evidence/conformance-v14-<hash>-<date>/`。

## 7. 风险与开放问题

### 7.1 WSL 下 Vulkan 呈现目标的可得性（本次实测）

**实测环境**：WSL2 Ubuntu + WSLg（`DISPLAY=:0`、`WAYLAND_DISPLAY=wayland-0`），Vulkan instance
1.4.357，ICD 三个（`lvp`、`dzn`、`nvidia`）。

**实测结论**（原始输出见 §8.6）：

- **instance 层齐全**：`VK_KHR_surface`(rev 25)、`VK_KHR_xcb_surface`、`VK_KHR_xlib_surface`、
  `VK_KHR_wayland_surface`、`VK_KHR_display`、`VK_EXT_headless_surface`、
  `VK_EXT_swapchain_colorspace` 都在。
- **两块设备都能做 swapchain**：`GPU id 0 = Microsoft Direct3D12 (NVIDIA GeForce RTX 5060)`
  （dzn）与 `GPU id 1 = llvmpipe (LLVM 22.1.8)` 都暴露 `VK_KHR_swapchain`(rev 70)；可呈现
  surface 类型为 `xcb`/`xlib` 与 `wayland`；格式只有 `B8G8R8A8_SRGB` 与 `B8G8R8A8_UNORM`；
  present mode 4 种；`minImageCount = 3`、`maxImageCount = 0`、`maxImageArrayLayers = 1`、
  `supportedTransforms` 只有 IDENTITY。
- **端到端可跑**：`vkcube --c 3` 在 llvmpipe（`--gpu_number 1`）与 dzn（`--gpu_number 0`）上都
  **退出码 0**；对照组 `env -u DISPLAY` 立即**退出码 1**
  （`Environment variable DISPLAY requires a valid value.n`），说明退出码 0 不是"根本没建
  surface"。这个报错文本末尾的 `n` 是 `vkcube` 自己输出的字面内容（已用 `od -c` 核对）。
- **纯 Linux 枚举看不到 NVIDIA 原生 ICD**：`nvidia_icd.json` 在 Linux 侧枚举不到设备
  （`evidence/render-feasibility-2026-09-14/README.md` 末段已记录），RTX 5060 原生驱动的数据
  来自在 WSL 里跑 Windows exe。**因此"WSL 下能呈现"必须写明是 dzn 还是 nvidia 原生。**

**风险**：本机可得 ≠ CI 可得（§1.2 第一条）。**结论：不必等真实 surface 才能做第一段，但也不能
把真实 surface 放进判据。**

### 7.2 `VK_KHR_swapchain` 与 headless 的矛盾

- `VK_EXT_headless_surface` 在 instance 层存在（§7.1），但 `vulkaninfo` 的 *Presentable Surfaces*
  一节只为 `xcb`/`xlib`/`wayland` 列出了 surface 类型与 present mode，**没有** headless 条目。
  **待确认**：headless surface 在这两块设备上是否 presentable
  （`vkGetPhysicalDeviceSurfaceSupportKHR` 是否为真），以及 present 是否退化成 no-op。
- 即使 headless surface 可用，**它也是一个 surface**：一旦启用 `VK_KHR_surface` +
  `VK_KHR_swapchain`，device 创建的扩展表就要变（`lib.rs:518-524` 现在只按需开
  `VK_EXT_external_memory_host`），并且要按 `vkGetPhysicalDeviceSurfaceSupportKHR` 重新规划
  family（§7.4）。这与"第一段不接 surface"的裁决冲突。
- **建议**：第一段**不启用** `VK_KHR_swapchain`；把"交换链等价物"做成 provider 自有的 image +
  显式所有权往返（§3.6）。headless surface 的可行性作为 §9 的一条待确认，留给接真实显示时再验。

### 7.3 交换链 image 的 tiling 与既有 optimal 附件路径的关系

- **好消息（tiling）**：既有路径已经是 `VK_IMAGE_TILING_OPTIMAL`（`render.rs:442`），且
  `admit_color_attachment` 是按 **optimal** 能力位判定的（`render.rs:411-431`，其结论来自
  `evidence/render-feasibility-2026-09-14/`：linear 在 NVIDIA/dzn 上不支持 `COLOR_ATTACHMENT`）。
  swapchain image 也是 optimal，**tiling 上不冲突**。
- **坏消息（格式）**：swapchain image 的格式由 surface 决定，本机实测只有 `B8G8R8A8_SRGB` 与
  `B8G8R8A8_UNORM`；而核心契约是闭集四值且**没有 sRGB**（`provider.rs:1107` 的注释就是这条
  裁决）。若要接真实 surface，就必须做一次"目标格式 ≠ 附件格式"的转换或新增 sRGB 变体：前者多
  一次 blit（破坏"present 是等价物"的简单性），后者推翻 `docs/23` §7.4。
- **建议**：第一段沿用 `Rgba8Unorm` 等价目标；把"surface 格式 → 契约格式"的映射列为接真实
  surface 时的首要待确认项（§9 第 4 条）。

### 7.4 presentation 队列族与 compute-only family 的关系

- **现状**：device 只创建"主 family + 一个 compute-only family"（`lib.rs:474-497`），总预算
  `MAX_DEVICE_QUEUES = 8`（`:72`）；`queue_families`（`:543-550`）与队列索引一一对应，
  `select_graphics_queue`（`render.rs:505-523`）只找 `GRAPHICS`，**从不查询 present 支持**。
- **真 swapchain 的硬要求**：graphics 与 present 必须存在 WSI 兼容对（同一 family，或该 family
  的队列支持 `vkGetPhysicalDeviceSurfaceSupportKHR`）。这要求 `select_physical_device` 增加
  surface 相关准入（`:888`）、family 计划增加一项、device 创建与 `MAX_DEVICE_QUEUES` 预算重算。
- **可观测后果**：真机判据里 `families` 与 `queues` 是被打印并归档的观测值（历史值
  `queues=8 families=2`，见 `docs/21:19-20`）。**新增 family 会改变这两个数**，必须同步更新归档
  判据与 `docs/21` 的 family 论述——这正是"第一段不新增 family"的第二个理由（第一个是 §7.2）。
- **待确认**：本机 dzn/llvmpipe 的 graphics family 是否同时支持 present（`vkcube` 的成功是间接
  证据，不是 `vkGetPhysicalDeviceSurfaceSupportKHR` 的直接读数）。

### 7.5 macOS native provider 需要的最小对称改动

按 §5.1 与 §4.2，最小集是四件事：（a）present 的值类型与 trace/codec 接线；（b）present 等价物
（`MTLTexture` + `usage = renderTarget` + 一次所有权往返，构造点参照
`crates/metal-api-native/src/render.rs:465-498`）；（c）`ProviderCapabilities` 的 present 位
（构造点 `native.rs:137-181`，默认关闭）；（d）oracle 侧的对应分支与 reviewed 白名单。

**前置条件**：native 的 `supports_render_passes`（`native.rs:181`）必须先翻转，而它的条件是
`--render-selftest` 在 Apple GPU 上通过（`conformance/RENDER-CAPTURE.md` §5）。
**待确认**：若坚持用 `CAMetalLayer`/`nextDrawable`，oracle 会引入 CoreAnimation 与窗口系统依赖，
在无窗口的 CI 上不可运行；这是一条**不建议**的路线，需要与用户确认是否真的要窗口级证据。

### 7.6 其它风险

- **错误传播**：不支持的目标格式、`image_count != 1`、present 与附件视图不一致、"目标所有权被
  别处持有"都需要新的 typed slug（现有形态见 `provider.rs:3927` 与 `docs/13`）；不要用泛化的
  `provider_unavailable` 掩盖。
- **count 契约漂移**：§5.3 的口径若与 v13 的派生规则不一致，会同时改动 `compare.py` 的强制分支
  与真机归档判据；必须一次说清"v14 起强制、v13 不变"。
- **hazard / lease 语义**：呈现目标以"整资源在途"起步（§3.1/§3.2），与 `docs/23` §3.6 同口径；
  subresource 级并发仍留到 range hazard（`docs/14`）之后。
- **工具顺序**：`tools/lavapipe-smoke.sh` 自动发现 `suite-v*.json`（`:25-30`），`suite-v14.json`
  必须在 capture 与比较器都支持之后再提交（`docs/18` 的原始阻断点一节的纪律）。
- **README 与文档漂移**：`1529f90` 刚刚把 README 的渲染范围改写准确（`README.md:128-141` 新增
  "Offscreen rendering executes on the Vulkan trace rail" 一段），并把未完成项收敛为
  `Not implemented: render passes beyond the Vulkan trace rail (no object-API render command
  encoder and no native-provider render path), sampler and texture generalisation beyond the
  sampled fixture, presentation and swapchain, ...`（`README.md:160-166`）。也就是说
  **presentation 已经写进同一条列表的中间**：presentation 落地后必须同步改写这一行（把
  "presentation and swapchain" 从列表里拿走或改写成准确的剩余范围），否则文档会继续误导。
- **objects-async 形状**：present 是否立即进入异步形状（`Submitted` → `wait` → `readback`）要
  显式决定；建议与渲染一样同步进 Step 5，避免后面再补一次五路径比较。

## 8. 抽样验证（`rg -n` 命令与原始输出）

以下命令在撰写时的 checkout 上执行，输出为原始粘贴（行号会随提交漂移，引用前重跑）。

### 8.1 队列族计划与 device 创建（只有两类 family）

```sh
cd /home/hiliang/hackintosh/metal-api-emulator
rg -n -B 2 -A 8 "let dedicated_compute" crates/metal-api-vulkan/src/lib.rs
rg -n "MAX_QUEUES_PER_FAMILY|MAX_DEVICE_QUEUES|queue_create_infos|create_device\(" crates/metal-api-vulkan/src/lib.rs
```

```text
472-            .clamp(1, MAX_QUEUES_PER_FAMILY);
473-        let mut family_plans = vec![(queue_family, primary_queues)];
474:        let dedicated_compute = queue_family_properties
475-            .iter()
476-            .enumerate()
477-            .filter(|(index, family)| {
478-                *index as u32 != queue_family
479-                    && family.queue_count > 0
480-                    && family.queue_flags.contains(vk::QueueFlags::COMPUTE)
481-                    && !family.queue_flags.contains(vk::QueueFlags::GRAPHICS)
482-            })
67:const MAX_QUEUES_PER_FAMILY: usize = 4;
72:const MAX_DEVICE_QUEUES: usize = 8;
472:            .clamp(1, MAX_QUEUES_PER_FAMILY);
487:                    (family.queue_count as usize).clamp(1, MAX_QUEUES_PER_FAMILY),
531:            .queue_create_infos(&queue_infos)
536:        let device = match unsafe { instance.create_device(physical, &device_info, None) } {
```

### 8.2 图形队列是"找出来的"，且不查询 present

```sh
rg -n -A 18 "fn select_graphics_queue" crates/metal-api-vulkan/src/render.rs
```

```text
505:fn select_graphics_queue(context: &VulkanContext) -> Result<usize, ProviderError> {
506-    let families = unsafe {
507-        context
508-            .instance
509-            .get_physical_device_queue_family_properties(context.physical)
510-    };
511-    context
512-        .queue_families
513-        .iter()
514-        .position(|family| {
515-            families
516-                .get(*family as usize)
517-                .is_some_and(|properties| properties.queue_flags.contains(vk::QueueFlags::GRAPHICS))
518-        })
519-        .ok_or_else(|| {
520-            capability_refusal("render_graphics_queue_unavailable")
521-                .with_detail("the selected device created no queue in a graphics-capable family")
522-        })
523-}
```

### 8.3 能力位与 admission 门

```sh
rg -n "supports_render_passes|max_color_attachments|max_attachment_dimension|supported_color_formats" crates/metal-api-core/src/provider.rs | head -8
rg -n -A 8 "fn admit_render_passes" crates/metal-api-core/src/provider.rs
```

```text
1242:/// grows the matching `ProviderCapabilities::max_color_attachments` field with
3647:    pub supports_render_passes: bool,
3651:    pub max_color_attachments: u32,
3654:    pub max_attachment_dimension: [u64; 2],
3659:    pub supported_color_formats: Vec<AttachmentFormat>,
3667:        self.supports_render_passes
3668:            || self.max_color_attachments != 0
3669:            || self.max_attachment_dimension != [0, 0]
3921:    fn admit_render_passes(&self, trace: &ComputeTrace) -> Result<(), ProviderError> {
3922-        let render_pass_count = trace.render_passes().count();
3923-        if render_pass_count == 0 {
3924-            return Ok(());
3925-        }
3926-        if !self.supports_render_passes {
3927-            return Err(capability_error("render_passes_unsupported")
3928-                .with_field("passes", FieldValue::Unsigned(render_pass_count as u64)));
3929-        }
```

### 8.4 五路径入口与 render plan 扩展点（presentation 不加 rail）

```sh
rg -n "ALLOCATION_OBSERVATIONS|def _render_plan|add_argument\(\"--" conformance/compare.py
```

```text
21:ALLOCATION_OBSERVATIONS = {
361:def _render_plan(plan, suite):
465:                 and all(isinstance(rail, str) and rail in ALLOCATION_OBSERVATIONS
521:    _require(isinstance(report["backend"], str) and report["backend"] in ALLOCATION_OBSERVATIONS,
525:    expected_observation = ALLOCATION_OBSERVATIONS[report["backend"]]
692:    parser.add_argument("--suite", required=True, type=Path)
693:    parser.add_argument("--check", type=Path, help="validate one capture; does not establish parity")
694:    parser.add_argument("--native", type=Path, help="reference capture produced by the Swift Metal collector")
695:    parser.add_argument("--vulkan", type=Path, help="capture produced by Vulkan")
696:    parser.add_argument("--metal-provider", type=Path,
698:    parser.add_argument("--vulkan-objects", type=Path,
700:    parser.add_argument("--metal-objects", type=Path,
```

### 8.5 零 present 代码（"从零开始"的直接证据）

```sh
rg -n -i "swapchain|VK_KHR_surface|create_surface|queue_present" crates examples
```

```text
（无输出；rg 退出码 1）
```

### 8.6 本机呈现可得性（`vulkaninfo` + `vkcube`，实测）

```sh
vulkaninfo --summary 2>&1 | head -12
vulkaninfo 2>/dev/null | rg -n "VK_KHR_surface|VK_KHR_swapchain|VK_KHR_xcb_surface|VK_KHR_wayland_surface|VK_EXT_headless_surface|^GPU id|Present Modes|PRESENT_MODE_|FORMAT_B8G8R8A8"
```

```text
WARNING: dzn is not a conformant Vulkan implementation, testing use only.
==========
VULKANINFO
==========

Vulkan Instance Version: 1.4.357

Instance Extensions: count = 27
-------------------------------
	VK_EXT_headless_surface                : extension revision 1
	VK_EXT_swapchain_colorspace            : extension revision 5
	VK_KHR_surface                         : extension revision 25
	VK_KHR_wayland_surface                 : extension revision 6
	VK_KHR_xcb_surface                     : extension revision 6
	VK_KHR_xlib_surface                    : extension revision 6
	VK_KHR_display                         : extension revision 23
81:GPU id : 0 (Microsoft Direct3D12 (NVIDIA GeForce RTX 5060)) [VK_KHR_xcb_surface, VK_KHR_xlib_surface]:
87:			format = FORMAT_B8G8R8A8_SRGB
90:			format = FORMAT_B8G8R8A8_UNORM
92:	Present Modes: count = 4
93:		PRESENT_MODE_IMMEDIATE_KHR
94:		PRESENT_MODE_MAILBOX_KHR
95:		PRESENT_MODE_FIFO_KHR
96:		PRESENT_MODE_FIFO_RELAXED_KHR
309:GPU id : 1 (llvmpipe (LLVM 22.1.8, 256 bits)) [VK_KHR_xcb_surface, VK_KHR_xlib_surface]:
938:	VK_KHR_swapchain                      : extension revision 70
2095:	VK_KHR_swapchain                                   : extension revision 70
```

```sh
cd /tmp
timeout 60 vkcube --c 3 --gpu_number 1 --wsi xcb > /tmp/vk-lvp-xcb.txt 2>&1; echo "exit=$?"; cat /tmp/vk-lvp-xcb.txt
timeout 60 vkcube --c 3 --gpu_number 0 --wsi xcb > /tmp/vk-dzn-xcb.txt 2>&1; echo "exit=$?"; cat /tmp/vk-dzn-xcb.txt
env -u DISPLAY timeout 30 vkcube --c 3 --gpu_number 1 --wsi xcb > /tmp/vk-ctl-nodisplay.txt 2>&1; echo "control_exit=$?"; cat /tmp/vk-ctl-nodisplay.txt
```

```text
exit=0
WARNING: dzn is not a conformant Vulkan implementation, testing use only.
Selected GPU 1: llvmpipe (LLVM 22.1.8, 256 bits), type: Cpu, apiVersion: 4211042 (1.4.354), driverVersion: 109060097 (26.2.1) 
exit=0
WARNING: dzn is not a conformant Vulkan implementation, testing use only.
Selected GPU 0: Microsoft Direct3D12 (NVIDIA GeForce RTX 5060), type: DiscreteGpu, apiVersion: 4202850 (1.2.354), driverVersion: 109060097 (26.2.1) 
control_exit=1
Environment variable DISPLAY requires a valid value.n
Exiting ...
```

## 9. 待确认清单（实现前必须钉死）

1. **present 的建模形状**（§3.5）：形状一（挂在 render pass 上的可选 `present`）还是形状二
   （trace 联合的第三个 pass）。这决定 §4.1 是"扩展 tagged 载荷"还是"新增 tag"，必须在 Step 2
   之前定死。
2. **真实 surface 的定位**（§7.1/§7.2）：第一段确认不接 `VK_KHR_swapchain`；本机
   `VK_EXT_headless_surface` 是否 presentable、以及 dzn/llvmpipe 的 graphics family 是否同时支持
   present（需要 `vkGetPhysicalDeviceSurfaceSupportKHR` 的直接读数，而不是 `vkcube` 的间接证据）。
3. **`Fifo` 的语义**（§3.3）：是"排队等一个可用目标"还是"立即完成一次目标轮转"；前者引入丢帧与
   超时，后者引入"present 可能失败但没有失败信号"。第一段建议后者 + 计数。
4. **surface 格式映射**（§7.3）：surface 只给 `B8G8R8A8_UNORM`/`B8G8R8A8_SRGB`，契约格式是闭集
   四值且无 sRGB。若将来接真实 surface，是"目标格式 ≠ 附件格式 + 一次 blit"，还是"新增 sRGB
   变体推翻 `docs/23` §7.4"。
5. **present/acquire 计数是否进 v14 强制契约**（§5.3）：v11/v12 的 `copy_in`/`copy_out` 是强制的
   （`compare.py:600-602`），v13 的渲染用例只做类型校验（`:595-599`）。present 计数若强制，会
   同时改 `compare.py` 与真机归档判据。
6. **对象 API 的 present 位置**（§5.2/§6 Step 5）：要不要与渲染一起进 Step 5（推荐），还是等
   render encoder 落地之后再补一次。
7. **Apple 侧的替代证据形式**（§5.5）：`--render-selftest` 式的一设备检查是否就是最终形式；以及
   `allocation_observation` 这个名称（`compare.py:21-26`）在出现"呈现目标"之后是否还合适（改名
   属于报告 schema 决策）。
