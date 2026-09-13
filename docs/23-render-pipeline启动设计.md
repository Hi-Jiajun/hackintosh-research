# render pipeline 启动设计：离屏单附件最小渲染（2026-09-13）

> 本文是图形/显示路径第二段（render pipeline → render pass 与附件 → presentation/swapchain）的
> 启动设计。`docs/16` 已经把这条序列钉成依赖序，并完成了第一段（纹理 + sampler 的 compute
> 采样，已进入 v11 五路径）；本文只定义**渲染轨道的第一段**：离屏 render pass + 顶点/片元
> pipeline，以及它进入五路径比较的最小口径。
>
> **边界**：本文只新增本文件，不写实现、不改任何代码仓、不 push。不改
> `docs/11-Metal-provider-contract-v0.md`（长期 dirty）、不改其它 docs。所有"现状"结论都在
> 撰写时实地读过代码，行号可用 §8 的三条 `rg -n` 复现；凡未在代码或既有文档中确认的行为，
> 一律标"待确认"，不臆造 API 名称。
>
> **路径约定**：`crates/**`、`conformance/**`、`examples/**` 相对
> `/home/hiliang/hackintosh/metal-api-emulator`（分支 `shared-provider-objects`，撰写时 HEAD
> `b692e97`）；`docs/NN-*.md` 相对 `/home/hiliang/hackintosh/research`（撰写时 HEAD `7820e48`）；
> pinned translator 相对 `~/.cargo/git/checkouts/metal2vulkan-ec622ce80a554d9d/43c46ac`
> （即 `Cargo.toml:23` 的 `rev = "43c46ac…"`）。
>
> **相关文档**：`docs/09` §13.3 第 5 条（图形路径未完成）、`docs/16`（图形路径规划：纹理优先）、
> `docs/18`（v11 接入检查清单）、`docs/14`/`docs/15`（range hazard 与共享分配）、`docs/13`
> （回收与设备丢失）、`docs/21`（队列优先级与 family）。
>
> **五路径标签**（`conformance/compare.py:20-26`，命令入口 `:469-482`）：`native-metal`（Swift
> oracle）/ `vulkan`（direct trace）/ `native-metal-provider`（Rust native trace）/
> `vulkan-objects` / `native-metal-provider-objects`。

## 1. 问题与目标

### 1.1 为什么先做 render pipeline 而不是 presentation

1. **观测面已经存在，且只对离屏成立。** 现有比较器只认两种 `allocation_observation`
   （`conformance/compare.py:20-26`）：Swift oracle 是 `gpu-buffer-readback`，其余是
   `host-writeback-landing`。两者都是"宿主可见字节"。离屏 render pass 天然落在这套语义里：
   渲染到附件 → 读回字节 → 与 fixture 的期望字节逐字节比较。presentation/swapchain 则要求
   全新观测面（present 时序、帧丢弃、图像获取/归还、宿主窗口系统对端），字节 parity 表达不了。
2. **presentation 依赖外部状态，不属于 provider 契约能独立验证的范围。** 宿主侧没有 Metal 的
   `CAMetalLayer` 对端，swapchain 必须对接宿主的窗口/显示系统，这属于 reims 与 display 轨道
   （见仓库根的 `HANDOFF-DISPLAY-*.md`），不是 `metal-api-core` 的中立契约。把它混进第一段会
   让"契约边界"和"宿主集成"两个风险同时打开。
3. **依赖序。** `docs/16` 开头钉的序列是：纹理 → sampler → 纹理读取的 compute 用例 →
   render pipeline（顶点/片元）→ render pass 与附件 → presentation/swapchain → heaps/ICB。
   render pass 与附件是 presentation 的前置：swapchain 的每一帧就是"渲染到可 present 的附件"。
4. **判别力。** 用 compute 写颜色冒充渲染无法证明顶点装配、viewport/裁剪、光栅化规则与附件
   store 语义——这些恰好是 render pipeline 的契约内容。第一段必须真的走图形管线。

### 1.2 第一个可验证里程碑（一句话）

**一个 2×2 的离屏单颜色附件 render pass：顶点着色器由 `vertex_id` 生成覆盖全屏的三角形，
片元着色器写固定颜色，附件在宿主侧读回的 4 个 texel 与 fixture 写死的期望字节逐字节一致，
并在五条路径上一致。**

### 1.3 里程碑的验收判据（可判定，不含主观项）

- **尺寸**：2×2（4 texel）。选 2×2 而不是 1×1，是为了让"三角形是否真的覆盖了整张附件"可证伪；
  1×1 无法区分"全覆盖"与"只写了 (0,0)"。
- **防伪**：附件初始字节由 fixture 预置哨兵（例如全 `0xFE`；v11 用的是首字 `0xFFFFFFFF`，
  先例见 `conformance/suite-v11.json`），因此"压根没执行也通过"不可能发生。
- **出口**：宿主可见字节比较，不看"画面是否好看"，不引入任何图像相似度指标。
- **明确不进第一段**：MSAA、深度/模板、多附件、动态状态（viewport/scissor 之外的 blend/cull/
  winding）、presentation/swapchain。

## 2. 现状核对

以下每条都能用 §8 的命令或同形 `rg -n` 复现。

### 2.1 `crates/metal-api-core/src/provider.rs`：pass/资源/能力位

- **pass 只有 compute 一种形状**：`pub struct ComputePass`（`:970`）字段是
  `pipeline` / `buffers: Vec<BufferView>` / `textures: Vec<TextureView>`（`:977`）/
  `dispatch: Dispatch`。**没有**附件、load/store、viewport、draw 顶点数或任何图形状态；整条
  trace 只有一个 dispatch 维度（`DispatchKind` `:809`、`Dispatch` `:824`）。
- **管线契约是 compute 契约**：`PipelineContract`（`:891`）只有 `dispatch_kind` /
  `required_local_size` / `fixed_grid` / push constant 范围 / `buffer_bindings` /
  `shader_capabilities` / `translator_revision`。**没有** stage 组合、入口对、顶点布局、附件格式。
- **纹理值已就位**：`TextureFormat`（`:551`）、`TextureType`（`:574`）、`TextureAccess`（`:600`）、
  `TextureSource`（`:615`）、`TextureView`（`:637`，含 `expected_bytes`/`validate_shape`）。
  附件可以复用 `TextureView` 的"形状 + 紧排 extent + 源三分支"表述，但需要新的 access/usage
  取值（当前只有 `Sampled`/`Storage`/`Unused`）。
- **能力位无渲染维度**：`ProviderCapabilities`（`:2773`）的字段全是 compute/内存相关
  （`max_passes`、`max_local_size`、`max_storage_buffer_descriptors`、`max_buffer_range`、
  `alias_mode`、`storage_modes`、`host_readback`、`submit_only`），**没有任何 render/attachment
  维度**。admission 入口是 `ProviderCapabilities::admit`（`:2813`）。
- **schema 与资源上限**：`PROVIDER_SCHEMA_VERSION = 2`（`:19`）、`MAX_SERIAL_RESOURCES = 64`
  （`:23`）；trace 自带 `schema_version: u16`（`:1076`），版本不符是 typed 拒绝（`:2246`）。
- **一处文档漂移（供后续顺手修正）**：`ComputePass.textures` 的注释（`:973-976`）仍写着
  "no provider executes them yet"，而实际从 v11 起两条 provider 路径都已执行纹理；注释早于
  实现（`docs/16` §4.6–§4.9）。
- **纹理拒绝已从 core 移到 provider**：`ContractError::TextureBindingUnsupported`（`:3892`）
  只剩定义与 slug/Display 映射（`:3100`、`:4072`），全仓没有构造点。
- **队列优先级不在设备对象里**：`QueuePriority`（`:3282` 附近）与 `select_queue_with_priority`
  （`:3456`）是宿主侧调度策略，与设备队列创建无关（`docs/21` §2）。

### 2.2 `crates/metal-api-vulkan/src/lib.rs`：pipeline/descriptor/queue

- **第一处硬门禁：反射校验要求 compute kernel。** `validate_pipeline_reflection`（`:1531`）在
  `:1536` 直接拒绝非 Kernel stage，并在 `:1555-1572` **显式拒绝** `argument_buffer_fields` /
  `vertex_attributes` / `varyings` / `render_targets` / `depth_members` / `depth_qualifier` /
  `stencil_members` / `vertex_builtins` / `tessellation` / imageblock 相关字段；`:1579` 进一步
  限定 binding kind 只能是 Buffer 或 Texture。渲染轨道要动的地方就是这里。
- **translator 早就具备渲染反射。** pinned translator 的 stage 枚举有 `Vertex`/`Fragment`
  （`43c46ac:src/passes/mod.rs:32`），`ShaderReflection` 带 `vertex_attributes`
  （`reflect/mod.rs:1873`）、`varyings`（`:1875`）、`render_targets`（`:1877`）、
  `depth_members`/`depth_qualifier`（`:1879`/`:1881`），并有 vertex/fragment 反射测试
  （`reflect/tests.rs:103`、`:875`）。也就是说：**渲染的缺口在 provider 的消费侧，不在翻译侧**
  ——这与 `docs/16` §4.4 侦察纹理时的结论形态一致。
- **描述符与执行资源已按 kind 分派**：`PoolKind`（`:854`）/`PoolKey`（`:860`）、
  `descriptor_type_for_binding`（`:1446`，纹理映射 `COMBINED_IMAGE_SAMPLER` `:1449`）、
  `ExecutionResources`（`:1893`，拥有 buffers/textures/descriptor pool/sets/command pool/
  command buffer/fence/queue_index）、`PipelineObjects`（`:1885`，创建于 `:2263`/`:2068`）。
- **队列：预算与 family 计划。** `MAX_QUEUES_PER_FAMILY = 4`（`:52`）、
  `MAX_DEVICE_QUEUES = 8`（`:57`）；`queue_family_count`（`:592`）是 distinct family 数。
  family 计划（`:404-431`）只由两项构成：**主 family**（选中的那个）与**一个 compute-only
  family**（`:412-425`，条件是"有队列 + 含 COMPUTE + 不含 GRAPHICS"）。`queue_families` 是
  与队列索引对齐的扁平表（`:482-486`，使用点 `:3010`）。
- **设备选择不要求 GRAPHICS。** `select_physical_device`（`:750-790`）的准入条件是 Vulkan 1.3
  `maintenance4`、`shaderInt8`、`shaderInt64` 与"至少一个含 COMPUTE 的 family"，打分时给含
  GRAPHICS 的 family 加分，但**不要求**。这解释了 §7.3 的结构性缺口。
- **同步现状**（只有 compute 时代的三处）：纹理镜像的 `PREINITIALIZED → GENERAL` 布局迁移
  （`:3032-3056`）、pass 之间的 compute→compute barrier（`:3085-3101`）、命令缓冲末尾的
  `COMPUTE_SHADER → HOST` 回读屏障（`:3131-3145`）。渲染需要新的
  `COLOR_ATTACHMENT_OPTIMAL` 路径，见 §7.1。

### 2.3 `crates/metal-api-native/src/`：macOS oracle 需要的对称面

- provider 只接受**六份审查过的 MSL fixture**，执行同步、限制在 Apple4 统一内存设备
  （`lib.rs:1-8` 头注释）。
- 编译路径直接取 `MTLComputePipelineState`（`native.rs:412-445`，`:435` 处 `msg_send!`）。
- 纹理支持已就位：`TextureDescriptor` → `MTLTextureType::D2` + `ShaderRead`（`native.rs:994-1001`），
  所有权在 `One owned MTLTexture`（`:734`）。
- 编码路径是 `MTLComputeCommandEncoder`（`:1045`）+ `set_compute_pipeline_state`（`:1052`）+
  `dispatch_threads`（`:1071`）。
- **没有任何 `MTLRenderPipelineDescriptor` / `MTLRenderCommandEncoder` / render pass 描述符。**
  `ProviderCapabilities` 的构造点（`:137-160`）也没有渲染能力位。

### 2.4 `conformance/`：比较器与 suite 的组织方式（扩展点在哪）

- **case schema（扩展点 A）**：`compare.py::_suite_plan`（`:84-160`）逐字段校验 suite JSON；
  v11 新增的 `textures` 段就在 `:106-130`（各自的 allocation、view、binding、format、access、
  `initial_hex` 与 extent 长度）。渲染用例要新增的是"附件"段与 draw 描述。
- **报告 schema（扩展点 B）**：`validate_capture`（`:309-345`）固定报告键集合
  （`schema_version`/`suite`/`suite_sha256`/`backend`/`allocation_observation`/`device`/
  `platform`/`results`），并规定 `provider_backend = backend != "native-metal"`（`:335`）——
  即 count 契约只对 provider 后端生效（`docs/15` §5b）。
- **count 契约**：v11 起强制 provider 上报 `copy_in`/`copy_out`（`:384-395`），且
  `copy_in = 本用例触及的 distinct allocation 数`（纹理各自算一个 allocation，`docs/18` 步骤 3）。
  渲染用例改不改这条口径，是本设计必须裁决的开放问题（§5.3、§7.6）。
- **capture 工具（扩展点 C）**：`examples/metal-smoke/src/bin/provider-capture.rs` 的 case 结构
  （`textures` 字段 `:136`）、直连轨的 `TextureView` 构造（`:1518-1595`）与对象 API 轨的
  `new_texture_with_bytes` + `set_texture`（`:1368-1399`）。渲染要在同一个二进制里增加第三条
  执行形状。
- **Swift oracle（扩展点 D，只有五路径比较需要）**：`NativeOracle.swift` 的 `CaseDefinition`
  （`:67`，含可选 `textures`）、`TextureDefinition`（`:82`）、`validateShape` 白名单
  （`:311`，v11 条目在 `:423`）、`validateTextures`（`:494`）、`MTLTexture` 构造（`:803-820`）。
- **工具链**：`tools/lavapipe-smoke.sh` 会自动发现 `suite-v*.json` 并跑全部轨道，因此**新 suite
  必须在工具支持之后才能落地**（`docs/18` 的原始阻断点一节把这条写成了显式纪律）。

### 2.5 现状小结（一句话）

纹理路径已经把"翻译 → 反射 → descriptor → 镜像上传 → 布局迁移 → dispatch → 回读 → 五路径
比较"整条链条打通；渲染轨道要复用的正是这条链条的**资源与比较骨架**，需要新造的只有
stage/入口组合、附件语义、图形管线对象与 draw 执行。

## 3. 契约设计

### 3.1 render pass 的最小字段集

第一版只允许**一个颜色附件**、**一次 draw**、**零顶点缓冲**。字段级清单：

| 字段 | 类型/取值 | 说明 |
|---|---|---|
| `pipeline` | `PipelineId` | 指向 trace 的 pipeline 表（沿用现有注册/释放流程） |
| `color` | `RenderAttachment` | 唯一附件，见下表 |
| `viewport` | `[u32; 4]` | 第一版只接受 `(0, 0, width, height)`：不是"支持动态 viewport"，而是把隐式值显式化 |
| `vertices` | `u32` | 第一版固定 3（全屏三角），不接受索引与实例化 |

`RenderAttachment`：

| 字段 | 类型/取值 | 说明 |
|---|---|---|
| `view_id` / `allocation_id` | `ViewId` / `AllocationId` | 与 buffer/texture 同一命名空间，复用 lease/range 语义 |
| `format` | `TextureFormat` 子集 | 第一版建议 `Rgba8Unorm` 或 `Bgra8Unorm`（**不含 sRGB**，理由见 §7.4） |
| `width` / `height` | `u64` | 与 `TextureView::expected_bytes` 同口径（紧排 texel 数） |
| `load` | `LoadOp::{Clear(color), Load, DontCare}` | 第一版只放行 `Clear` 与 `Load`；`Clear` 让期望字节可写死 |
| `store` | `StoreOp::{Store, DontCare}` | 第一版只放行 `Store`（否则无法比较） |
| 宿主侧落地 | 复用 `TextureView` 的 source 三分支 | 具体口径见 §3.5 |

### 3.2 pipeline 描述

第一版 pipeline 是**一对入口 + 一种附件格式**，不加任何可选图形状态：

| 字段 | 取值 | 说明 |
|---|---|---|
| `vertex_entry` | 函数名 | 与 `fragment_entry` 属于同一个 `FunctionIdentity`（同一份源），或两条独立 compile 请求（待确认） |
| `fragment_entry` | 函数名 | 同上 |
| `color_format` | 与附件 `format` 相同 | 不匹配必须在 admission 阶段 typed-refuse，而不是交给驱动 |
| `vertex_layout` | `VertexLayout::Empty` | 顶点位置由 `vertex_id` 生成，**不引入 `MTLVertexDescriptor` 与顶点缓冲反射** |
| `blend`/`cull`/`winding`/`depth`/`stencil`/`sample_count` | 不存在 | 用"字段不存在"表达"不支持"，避免默认值歧义 |

### 3.3 明确不做（第一版之外，带触发条件）

| 不做项 | 理由 | 什么时候做 |
|---|---|---|
| MSAA（sample count > 1） | `TextureType` 已有 `is_multisample` 谓词，但 resolve 语义与比较口径都要新设计 | presentation 之后，或真实 guest shader 需要时 |
| 深度/模板附件 | translator 已反射 `depth_members`/`depth_qualifier`/`stencil_members`，但格式矩阵与比较口径翻倍 | 有深度用例的实际需求时 |
| 多颜色附件 / MRT | `render_targets` 是 `Vec`，附件模型要变成数组 + location 映射 | 单附件跑通并进入 v12 之后 |
| 索引/实例化 draw | 引入索引缓冲、实例步进与更多顶点布局 | 顶点缓冲进来之后 |
| 顶点缓冲与 `MTLVertexDescriptor` | 反射有 `vertex_attributes`，但需要新的资源类别与 footprint 证明 | §6 Step 7 之后单独一段 |
| 动态状态（blend/cull/scissor） | 每一种都需要独立的真机 parity 证据 | 按需 |
| presentation/swapchain | §1.1 的四条理由 | 另立轨道（宿主显示路径） |
| heaps / ICB | `docs/16` 序列的末端 | 另立轨道 |

### 3.4 类型草图（字段级，不是可编译代码）

```text
enum LoadOp { Clear(u32), Load, DontCare }
enum StoreOp { Store, DontCare }

struct RenderAttachment {
    view_id: ViewId, allocation_id: AllocationId,
    format: TextureFormat, width: u64, height: u64,
    load: LoadOp, store: StoreOp,
}

struct RenderPass {
    pipeline: PipelineId,
    color: RenderAttachment,
    viewport: [u32; 4],   // 第一版只接受 (0,0,w,h)
    vertices: u32,        // 第一版固定 3
}

// ComputeTrace 的 passes 变成带判别位的联合：
enum Pass { Compute(ComputePass), Render(RenderPass) }

struct RenderPipelineContract {
    vertex_entry: String, fragment_entry: String,
    color_format: TextureFormat,
    vertex_layout: VertexLayout,   // Empty（第一版唯一取值）
}
```

### 3.5 附件在宿主侧的落地方式（必须先定的实现口径）

比较器只看宿主可见字节（§1.1），所以附件的"落地"方式直接决定实现代价与 count 语义。三条候选：

1. **host-visible 线性 image 直接作附件**（与 v11 的纹理镜像对称）。优点：零新增 copy 语义、
   回读路径与纹理一致；风险：`VK_IMAGE_TILING_LINEAR` 能否作颜色附件依赖实现（§7.2）。
2. **optimal tiling 附件 + `vkCmdCopyImageToBuffer` 到 host-visible buffer**。优点：附件布局
   合规性最好；代价：多一次 image→buffer copy，count 契约要显式扩展（§5.3）。
3. **optimal tiling 附件 + 独立 staging 池**。与候选 2 同类，只是把 staging 复用；复杂度最高，
   第一版不建议。

本设计的**建议**：先按候选 1 做，在 Lavapipe 与 RTX 5060 上各测一次（§6 Step 4 的验收）；
若任一设备拒绝 linear tiling 附件，再退到候选 2，并把 count 口径的改动写进 §5.3 的契约。
**待确认**：这一点必须实测，不能靠文档推断。

### 3.6 附件与既有资源语义的关系

- 附件走"整资源在途"预约起步，与纹理第一版同口径（`docs/16` §4.1）；subresource 级并发留到
  range hazard（`docs/14`）之后。
- 附件与 buffer 共享同一 lease/allocation 命名空间，因此 hazard 判定与写回合并规则不需要新机制。
- `StoreOp::DontCare` 在第一版被拒，是为了让"没有落地"不能伪装成"落地正确"。

## 4. 线格式与能力位

### 4.1 `MCC1` 需要新增什么

- **复用而非新造通道**：沿用 `crates/metal-api-ipc/src/command_codec.rs` 的 `MCC1` 帧
  （`COMMAND_FRAME_MAGIC` `:26`），trace 编码入口 `put_trace`（`:1263`）/ `get_trace`（`:1288`）。
  渲染 pass 作为 pass 的一个分支进入同一帧，不新开命令种类——这样 `compile`/`submit`/`wait`/
  `readback`/`cancel`/`release` 的既有生命周期与 lease 语义零改动（`docs/13`、`docs/09` §13.3 第 1 条）。
- **payload 增量**：每个 pass 前置一个 kind 判别；kind=render 时追加附件字段（view/allocation/
  format/extent/load/store）、viewport 与顶点数；pipeline 表条目追加 vertex/fragment 入口与附件
  格式。字段顺序一次定死，避免后续再改版本。
- **不新增**：不改 magic、不改 completion/lease/borrowed 传输、不改 descriptor 通道。

### 4.2 能力位如何声明

与 `AliasMode`/`StorageMode` 的既有风格一致：**显式声明、默认关闭、旧 provider 行为不变**
（`docs/16` §4.1 对纹理给的也是同一裁决）。

- 建议增补：`supports_render_passes: bool`、`max_color_attachments: u32`（第一版 1）、
  `max_attachment_dimension: [u64; 2]`、`supported_color_formats: Vec<TextureFormat>`。
- 拒绝路径：core 的 admission（`:2813`）按能力位 typed-refuse，provider 在反射/管线创建阶段
  再拒一次——现有纹理路径就是这两层（`docs/16` §4.2 与 §4.7）。
- 明确**不**把"是否支持 GRAPHICS 队列"写进中立契约（那是设备侧事实），改成"渲染用例能否落在
  这个设备上"的能力位（§7.3）。

### 4.3 旧 trace 解码兼容策略

- `PROVIDER_SCHEMA_VERSION` 从 2 升到 3（`provider.rs:19`）；`get_trace` 目前是**无条件顺序
  解码**（`command_codec.rs:1288-1336`），因此必须先把 `schema_version` 读出来并**分支解码**：
  版本 1/2 走现有字段序列，版本 3 走新序列。
- 未知版本沿用 `ContractError::UnsupportedSchemaVersion` 的 typed 拒绝（`provider.rs:2246`，
  回归测试形态见 `:5709`），不做"尽力猜测"。
- **回归判据**：v1–v11 的 suite 在变更后仍字节一致通过（`compare.py --check` + 既有 capture
  计数），并新增一条 render pass 的 IPC 往返用例（先例：纹理往返，`docs/16` §4.2 第 2 步）。
- **不要做**：不要让"文本 IR 默认、v8 raw/wrapped"的 `air_encoding` 维度与版本升级互相放大
  （§6 Step 2 的验收里显式跑 v8）。

## 5. 五路径比较方式

### 5.1 Swift oracle 需要什么（`native-metal`）

原生参考实现要按真实 Metal 面写，最小集：

- `MTLRenderPipelineDescriptor`：`vertexFunction` / `fragmentFunction`（同一 `MTLLibrary` 的两个
  入口）、`colorAttachments[0].pixelFormat` 与附件一致；
- `MTLRenderPassDescriptor`：`colorAttachments[0]` 的 `texture` / `loadAction` / `storeAction`
  对应契约的 `load`/`store`；
- `MTLTexture` 作为附件：`usage` 必须含 `renderTarget`——而当前 oracle（`NativeOracle.swift:803-820`）
  与 native provider（`native.rs:994-1001`）都是**只读采样**的用法，需要新增分支；
- `MTLRenderCommandEncoder`：`setRenderPipelineState` + `drawPrimitives`（全屏三角）；
- 完成后 `getBytes` 回读，并沿用 oracle 既有的 guard/只读缓冲检查与 `validateShape` 白名单风格
  （`:311`、`:423`）。

### 5.2 Vulkan 三条轨（`vulkan` / `vulkan-objects` / `vulkan-objects-async`）怎么捕获

- 执行侧：`VkRenderPass` + `VkFramebuffer` + 图形 `VkPipeline`（vertex+fragment 两段 SPIR-V）+
  附件 image/view + `vkCmdDraw(3,1,0,0)`；附件资源的创建/释放并进 `ExecutionResources`
  （`:1893`）的所有权集合，命令缓冲、fence、queue 索引沿用现状。
- 与现有 buffer 契约的**真实差异**：现有观测面是 storage buffer 的 host writeback（`copy_out`），
  附件是 image → 需要一条"image → 宿主字节"的落地路径（§3.5），并因此在报告里多一个 allocation
  （附件）与对应 count 项。比较器一侧不需要新观测面：仍然是字节，仍然每个 allocation 只比较
  一次（`compare.py:450` 附近的唯一化规则）。
- 捕获工具：`provider-capture.rs` 需要第三条执行形状（现有两条：`:1518-1595` 直连、
  `:1368-1399` 对象 API）。对象 API 轨还需要 core 侧新增 render encoder——现有只有
  `compute_command_encoder`（`provider_api.rs:699`）与 `ComputeCommandEncoder`（`:1061`）。

### 5.3 计数契约（v11 之后的新增维度）

- 现行规则：`copy_in = 本用例触及的 distinct allocation 数`，v11 因为纹理各算一个 allocation
  而变成 2（`docs/18` 步骤 3；`compare.py:384-395`）。
- 渲染用例的**建议**口径：附件是 distinct allocation，`copy_out` 计 1（它被写并被读回）；
  `copy_in` 视 `LoadOp` 而定——`Clear` 不需要上传（0），`Load` 需要预置字节（1）。
- **待确认**：这条口径要不要写进 v12 的强制契约（v11 是强制的，`compare.py:384`）。建议强制，
  否则"附件有没有真的被 GPU 写过"缺少一个可判定的旁证。

### 5.4 Rust native provider 两轨（`native-metal-provider` / `-objects`）

- trace 轨与对象 API 轨都要接：`MTLRenderPipelineState` + `MTLRenderCommandEncoder`；设备准入
  沿用 Apple4 统一内存判定（`native.rs:104-160`）。
- provider 仍只接受**审查过的 fixture**（`lib.rs:1-8`），因此要新增第 7 份 reviewed MSL 并更新
  白名单——先例是 `docs/18` 最终状态里"native 白名单缺 `read_texture_2d` → 以 reviewed source
  登记"这条修复。

### 5.5 哪些断言先不要求

- presentation 时序、帧同步（跨队列 semaphore/fence 语义）、丢帧与 present 模式；
- MSAA resolve 精度、深度/模板比较函数、混合方程；
- sRGB 编码路径（第一版用非 sRGB 8-bit unorm，§7.4）；
- 任意 viewport/scissor、多附件、动态状态；
- 顶点缓冲 / 索引 / 实例化；
- 真实窗口与显示输出（属于 display 轨道，不在 provider 契约内）。

## 6. 实施步骤（每步独立可验证）

**Step 1 — core 纯类型与校验。** 新增 `RenderAttachment`/`LoadOp`/`StoreOp`/`RenderPass`/
`RenderPipelineContract` 值类型 + 校验（零维度、格式与附件不匹配、viewport 与 extent 不符、
`Clear` 颜色长度与格式不符、`store = DontCare` 第一版拒绝）+ 单测。
*验收*：新增单测通过；全仓行为零变化（v1–v11 的 capture 与测试计数不变）。先例：`docs/14`
Step 1 与 `docs/16` §4.2 第 1 步都是"纯类型先行、行为零变化"。

**Step 2 — trace 与线格式。** `ComputePass` 扩成带 kind 的 pass；`MCC1` 的 trace
`schema_version` 升到 3 并做版本分支解码；IPC 往返测试。
*验收*：Lavapipe 全套件（v1–v11，含 v8 raw/wrapped 编码）与本机全门禁不变；新增 render pass
往返用例通过；未知版本仍 typed-refuse。

**Step 3 — Vulkan 反射门放开。** `validate_pipeline_reflection`（`:1531`）接受 Vertex/Fragment
stage 与 `render_targets`/`varyings` 白名单，创建图形管线（render pass 对象、两段 SPIR-V、
附件格式）。
*验收*：一个最小 vertex+fragment fixture 在 Lavapipe 上**管线创建成功**（断言测试，先不执行）；
不在白名单内的反射（tessellation/imageblock/深度）仍是明确拒绝。

**Step 4 — Vulkan 直连执行。** 附件 image/view、framebuffer、`vkCmdDraw`、附件 → 宿主字节落地
（先按 §3.5 候选 1，失败退候选 2）、必要的布局迁移与 barrier。
*验收*：Lavapipe 上 2×2 用例的 4 个 texel 与 fixture 期望字节逐字节一致；哨兵未被保留；
`provider-capture` 的 direct 轨报告能过 `compare.py --check`。

**Step 5 — 对象 API 与第三条 Vulkan 轨。** core 侧 render encoder（与 `ComputeCommandEncoder`
`:1061` 对称）、`provider-capture.rs` 的 objects / objects-async 形状。
*验收*：三条 Vulkan 轨在同一条用例上字节一致；对象 API 的 commit/wait/readback 生命周期
（含 cancel 与 deadline）沿用现有回归。

**Step 6 — native 对称（含 Swift oracle 的渲染分支）。** `MTLRenderPipelineState` +
`MTLRenderCommandEncoder` + `MTLTexture(usage=renderTarget)`；新增第 7 份 reviewed MSL；
oracle 侧按 §5.1 扩展。
*验收*：macOS/Apple Paravirtual 上 `native-metal` / `native-metal-provider` /
`native-metal-provider-objects` 三轨一致（先于五路径整体比较）。

**Step 7 — conformance v12 与五路径。** `suite-v12.json`（2×2 附件 + 全屏三角 fixture）、
`compare.py` 支持附件段、`tools/lavapipe-smoke.sh` 的自动发现顺序**严格后置**、Swift oracle
渲染用例接入后打开五路径比较。
*验收*：Lavapipe 全套件绿；RTX 5060 真机三轨捕获 + `--check` 全 PASS；五路径
`compare-captures` 一致；证据按惯例归档 `evidence/conformance-v12-<hash>-<date>/`。

## 7. 风险与开放问题

### 7.1 附件布局与转换（`VK_IMAGE_LAYOUT`）

现状：纹理镜像走的是 `PREINITIALIZED → GENERAL`（`:3032-3056`，因为采样用不上
`SHADER_READ_ONLY_OPTIMAL`），回读只在末尾插一次 `COMPUTE_SHADER → HOST`（`:3131-3145`）。
渲染附件必须经过 `COLOR_ATTACHMENT_OPTIMAL`，且必须显式给出 `initialLayout`/`finalLayout`
（否则提交时校验层报错、真机上表现为内容未定义）。第一版需要的最短链条：
`PREINITIALIZED|UNDEFINED → COLOR_ATTACHMENT_OPTIMAL`（写）→ `GENERAL`（宿主可见/拷贝源）→
`HOST_READ` 可见性屏障。**待确认**：`StoreOp::Load` 与 `Clear` 组合时，
`VK_ATTACHMENT_LOAD_OP_LOAD` 是否需要额外初始化迁移——`PREINITIALIZED` 的语义是"内容未定义"，
不是"内容保留"。

### 7.2 tiling 与 render pass 兼容性

`VK_IMAGE_TILING_LINEAR` 的 host-visible image 能否作为颜色附件，取决于实现的
`VK_FORMAT_FEATURE_COLOR_ATTACHMENT_BIT` 与 linear tiling 的组合；这是 §3.5 候选 1 与候选 2 的
分水岭。**待确认**：Lavapipe（`llvmpipe`）与 RTX 5060 驱动两边都要实测；两边结论不一致时，
应选**都能过**的那条（大概率是候选 2），并接受多一次 image→buffer copy。

### 7.3 presentation 队列族与 compute-only family 的关系

- 现状：设备只创建"主 family + 一个 compute-only family"（`:404-431`），总预算
  `MAX_DEVICE_QUEUES = 8`（`:57`）；`queue_families`（`:482-486`）与队列索引一一对应，
  真机判据里 `families` 与 `queues` 是**被打印并归档**的观测值（`provider_suite.rs:2478`、
  `:2600`、`:2715`；真机历史值 `queues=8 families=2`，见 `docs/21:18-20`）。
- 因此：**渲染第一段不应新增 family**。`vkCmdDraw` 必须落在含 GRAPHICS 的 family 上，而现在的
  device 只是**可能**创建了这样的 family（主 family 恰好含 GRAPHICS 时才行；`:750-790` 不保证）。
  第一版应把"该设备是否存在可用的 graphics family"表述为能力位，缺少时 typed-refuse。新增
  family 会改变 `families`/`queues` 的观测值，需要同步更新归档判据与 `docs/21` 的 family 论述。
- presentation 是另一件事：swapchain 要求 graphics family 与 present family 存在 **WSI 兼容对**
  （同一 family，或支持平台 present 的 family）。这既涉及设备选择条件的改动，也涉及 reims/display
  轨道，**不属于本段**；本段只需保证"附件 → 宿主字节"不依赖任何 present 队列。**待确认**：若
  将来复用 compute-only family 提交图形工作，驱动可能直接拒绝（family 的 `queueFlags` 不含
  GRAPHICS），必须在 admission 之前判定。

### 7.4 sRGB 与格式匹配

比较器是**字节** parity（§1.1），所以"格式对了但编码不同"会变成系统性偏差：Metal 侧
`.bgra8Unorm_srgb` 的写入会做线性→sRGB 编码，Vulkan 侧若用非 sRGB 格式，字节就会不同；两边
都用 sRGB 格式时，`Clear` 颜色的解释也要求一致（clear 值是否被视为线性）。第一版**建议完全
不碰 sRGB**：只用非 sRGB 的 8-bit unorm（`Rgba8Unorm`/`Bgra8Unorm`），把 sRGB 与色彩空间匹配
留到有真实 guest shader 需求时再单独立项。**待确认**：Metal 的 `MTLPixelFormat.bgra8Unorm` 与
Vulkan `VK_FORMAT_B8G8R8A8_UNORM` 的通道顺序在附件写入路径上是否严格同构——这需要一次真机
对照，不能从文档推断。

### 7.5 macOS native provider 需要的最小对称改动

按 §5.1/§5.4，native 侧最小改动是四件事：（a）render pass 的值类型与 trace 接线；
（b）`MTLRenderPipelineState` 创建路径（现在是 `MTLComputePipelineState`，`native.rs:412-445`）；
（c）`MTLRenderCommandEncoder` 编码路径（现在是 `MTLComputeCommandEncoder`，`:1045-1071`）；
（d）第 7 份 reviewed MSL fixture 与白名单（`lib.rs:1-8`）。此外 `ProviderCapabilities` 的构造点
（`:137-160`）要加 §4.2 的渲染能力位，默认关闭以保持旧行为。**待确认**：oracle 侧的
guard/只读检查（目前围绕 MTLBuffer）是否需要为附件 MTLTexture 定义等价检查（例如第二个只读
附件或纹理周边区域的 sentinel）。

### 7.6 其它风险

- **hazard/lease 语义**：附件以"整资源在途"预约起步（§3.6），与纹理第一版同口径。
- **count 契约漂移**：§5.3 的口径若与 v11 不一致，会同时改动 `compare.py` 的强制分支与真机
  归档判据；改动必须一次说清"v12 起强制、v11 不变"。
- **错误传播**：不支持格式、缺少 graphics family、framebuffer 创建失败都需要新的 typed slug
  （现有形态见 `provider.rs:3098-3102`、`docs/13`）；不要用泛化的 `provider_unavailable` 掩盖。
- **工具顺序**：`tools/lavapipe-smoke.sh` 自动发现 `suite-v*.json`，`suite-v12.json` 必须等到
  capture 与比较器都支持之后再提交（`docs/18` 原始阻断点一节的纪律）。
- **objects-async 轨**：渲染是否立即进入异步轨（对象 API 的 `Submitted` → `wait` → `readback`）
  需要显式决定；建议同步进 Step 5，避免后续再补一次五路径比较。

## 8. 抽样验证（`rg -n` 命令与原始输出）

以下命令在撰写时的 checkout 上执行，输出为原始粘贴。

### 8.1 反射门：渲染字段被显式拒绝

```sh
cd /home/hiliang/hackintosh/metal-api-emulator
rg -n "is not a compute kernel|outside the Phase 1 buffer-compute subset|only Metal buffers and sampled textures" crates/metal-api-vulkan/src/lib.rs
```

```text
1536:        return Err(failure("pipeline reflection is not a compute kernel"));
1572:            "pipeline uses Metal resources or specialization state outside the Phase 1 buffer-compute subset",
1579:                "Phase 1 supports only Metal buffers and sampled textures, not {:?}",
```

### 8.2 core 的 pass 形状（无附件、无 draw）

```sh
rg -n -A 11 "pub struct ComputePass" crates/metal-api-core/src/provider.rs
```

```text
970:pub struct ComputePass {
971-    pub pipeline: PipelineId,
972-    pub buffers: Vec<BufferView>,
973-    /// Texture bindings for this pass. The first texture increment carries the
974-    /// values through the trace and the command channel, but no provider
975-    /// executes them yet: admission refuses a non-empty list with a typed
976-    /// capability error (`research/docs/16` §4.2).
977-    pub textures: Vec<TextureView>,
978-    pub dispatch: Dispatch,
979-}
980-
981-impl ComputePass {
```

### 8.3 比较器的纹理扩展点与五路径入口

```sh
rg -n "textures = _list|texture_allocations.add|vulkan-objects|metal-objects" conformance/compare.py
```

```text
106:        textures = _list(case.get("textures", []), f"{where}.textures")
127:            texture_allocations.add(allocation)
471:    parser.add_argument("--vulkan-objects", type=Path,
473:    parser.add_argument("--metal-objects", type=Path,
480:                         "--vulkan-objects, or --metal-objects")
498:                                  (args.vulkan_objects, "vulkan-objects"),
```

### 8.4 附带两条（translator 与队列预算）

```sh
cd ~/.cargo/git/checkouts/metal2vulkan-ec622ce80a554d9d/43c46ac
rg -n "pub enum Stage" -A 6 src/passes/mod.rs
rg -n "pub render_targets|pub vertex_attributes|pub depth_qualifier" src/reflect/mod.rs
```

```text
32:pub enum Stage {
33-    Vertex,
34-    Fragment,
35-    /// Metal compute kernel (`!air.kernel`). GLCompute entry, LocalSize (64,1,1) default, compute
36-    /// thread/grid builtins -> Vulkan builtins or local-size constants, `air.buffer` -> SSBO.
37-    Kernel,
38-}
1873:    pub vertex_attributes: Vec<VertexAttribute>,
1877:    pub render_targets: Vec<RenderTarget>,
1881:    pub depth_qualifier: Option<crate::meta::DepthQualifier>,
```

```sh
cd /home/hiliang/hackintosh/metal-api-emulator
rg -n "MAX_DEVICE_QUEUES|dedicated_compute" crates/metal-api-vulkan/src/lib.rs | head -3
```

```text
57:const MAX_DEVICE_QUEUES: usize = 8;
412:        let dedicated_compute = queue_family_properties
428:        if let Some(plan) = dedicated_compute {
```

## 9. 待确认清单（实现前必须钉死）

1. **linear tiling 附件可用性**（§3.5/§7.2）：Lavapipe 与 RTX 5060 各测一次；决定候选 1 还是候选 2，
   并把 count 口径的后果写进 §5.3。这是唯一会改变实现形状的未知项。
2. **附件的 count 契约**（§5.3）：`copy_in`/`copy_out` 的取值与是否在 v12 强制；以及
   `LoadOp::Clear` 是否真的不需要 `copy_in`。
3. **render pass 对象形态**（§5.2/§7.1）：`VkRenderPass` 兼容对象还是 dynamic rendering；若用后者，
   需要新增 feature/扩展门与 `select_physical_device` 的准入条件。
4. **顶点/片元两个入口的承载方式**（§3.2）：同一 `FunctionIdentity` 的两个函数，还是两条 compile
   请求；这决定 pipeline 表条目与 `CompiledComputePipeline` 的字段是否要扩展。
5. **oracle 的附件校验口径**（§7.5）：MTLTexture 的 sentinel/guard 检查是否引入；以及
   `native-metal` 的 `allocation_observation` 是否仍写 `gpu-buffer-readback`（这个名称已不能覆盖
   纹理/附件，是否改名属于报告 schema 决策）。
6. **MCC1 v3 的最终字段顺序**（§4.1）：定死后不再回改，避免第二次版本升级。
