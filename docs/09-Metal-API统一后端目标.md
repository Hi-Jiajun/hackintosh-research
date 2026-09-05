# Metal API 统一后端目标与防偏航说明

> 状态：Proposed / local decision，2026-09-05。
>
> 这是一份本地架构基线，不代表 upstream 已接受，也不代表生产路径已经切换。
> 它依据用户提供的 steel-brain Discord 原文整理；Discord 原始频道不在本仓库内。
> 代码能力和完成度只以可复现的测试、parity 证据和 VM 验收为准。

## 1. 目标

长期目标不是维护两套独立的 vGPU 语义模型，而是：

```text
guest app
  -> guest Metal.framework
  -> AppleParavirtGPU.kext
  -> vGPU wire
  -> reims backend-neutral guest model
  -> canonical host Metal semantic execution path
  -> host Metal API provider
       ├─ macOS: native Metal provider
       └─ Windows: Vulkan-backed Metal API emulator
```

这里的“canonical Metal rail”只表示 host GPU API 语义和调用序列的参考实现。
它不移动已经确定归属的 neutral owner：

- wire/tag 含义仍归 `reims-vgpu-protocol`；
- transaction、dependency、publication 和调度仍归 `reims-vgpu-core` / runtime；
- guest RAM、page footprint 和导入仍归 `reims-vgpu-memory`；
- QEMU ABI、显示壳和窗口不因本目标自动迁移。

Windows provider 的职责是实现同一组 Metal-operation contract，然后用 Vulkan
执行。direct Vulkan rail 在迁移期间继续作为现状/控制路径；只有某个类别完成
provider parity 并接入 canonical path 后，才逐类移除重复语义。

## 2. Discord 原意的准确解读

steel-brain 的消息包含四个具体判断：

1. 先把 **Metal-on-Metal rail** 做成可信参考路径，而不是继续扩张独立的
   Metal rail / Vulkan rail 两套业务语义。
2. Windows 需要的是实际 Metal API 的 emulator，不是只有 AIR/shader translator。
3. 相同的简单 Metal 程序应能在 native Metal 和 emulator 下离线运行，缩短修改到
   测量的周期，不再每次依赖长时间 VM boot。
4. 当前 Vulkan 路径同时处理 vGPU API、AIR→SPIR-V 和 Metal/Vulkan 差异，因此一个
   黑块或同步错误可能来自多层；provider 抽象让错误归因可分层。

这不是“马上让任意 macOS Mach-O 应用在 Windows 加载假的 `Metal.framework`”，也
不是把 Darling 方案并入 reims。它是 reims 宿主侧的 Metal API provider 计划。

## 3. 层次和名称

| 层 | 真实职责 | 本计划中的位置 |
|---|---|---|
| guest `Metal.framework` | VM 内真实 Apple API/runtime，生成 AppleParavirtGPU 命令 | 不替换、不修改 |
| host macOS Metal provider | reims Metal rail 对 Apple `metal` crate/Metal runtime 的调用 | 参考实现 |
| host Windows Metal emulator | 实现 provider contract，内部使用 Vulkan | 主要新工作 |
| `metal2vulkan` | AIR/LLVM IR → SPIR-V | 独立 shader translator |
| reims Vulkan rail | 现有兼容/测量路径，直接处理 Vulkan | 迁移期控制路径 |

“Metal API emulator”在本文中是 host provider，不等于完整二进制兼容的
`Metal.framework`。

## 4. 现状和目标形态

### 4.1 现状

```text
vGPU command
  ├─ Metal backend -> host Metal API
  └─ Vulkan backend -> Vulkan API
                       ├─ AIR -> SPIR-V
                       └─ Metal/Vulkan semantic fixes
```

两条 backend 都拥有部分执行语义，差异可能同时来自 decoder、translation、resource
state 和 host API mapping。

### 4.2 目标

```text
vGPU command
  -> neutral decode / transaction / memory owners
  -> canonical Metal semantic execution
  -> provider
       ├─ native Metal
       └─ Vulkan emulator
```

provider 不重新解包 vGPU packet，也不拥有 guest scheduler 或 guest RAM lifetime。

## 5. 不可改变的合同

### 5.1 语义唯一归属

- canonical Metal path 是 host API semantic reference，不是新的协议解释器；
- emulator 不复制 vGPU decoder，不复制 neutral transaction model；
- `metal2vulkan` 不加入 `MTLDevice`、queue、resource 或 command-buffer 对象模型；
- 当前 AGENTS.md 的 ownership 和双 rail 规则仍有效。本文是迁移 overlay：在逐类
  parity 前不得删除现有 Vulkan 路径，也不得用实验开关让同一 packet 在两套模型中
  同时产生副作用。

### 5.2 provider 对称性

相同的 API trace 应可交给 native Metal provider 或 Vulkan provider。比较的是可观察
语义：输出 bytes/pixels、completion、错误类别、资源可见性和 hazard 结果；不要求
内部句柄、内存布局或缓存形状相同。

### 5.3 生命周期和完成

- provider 必须保留输入到 GPU 完成；
- 结果可见后才能发布 completion；
- allocation、compile、encode、submit、wait、readback 都必须返回结构化 refusal；
- “5 秒 fence wait”只表示已提交 fence 的等待上限，不是整个调用的 deadline；
- 线程模型、句柄可发送性、no-copy buffer 地址稳定性必须写入 provider contract。

## 6. 当前成果：原型，不是生产切换

本地原型仓库位于本机 workspace 的 sibling 目录（尚未作为本研究仓库的子目录发布）。
其当前提交和构建边界记录在 [upstream-69 适配证据](../contrib/evidence/metal-api-emulator-upstream69-2026-09-05.md)。

已完成：

- `metal-api-core`：源码级 Device/Library/Function/Pipeline/Queue/CommandBuffer/
  ComputeEncoder/Buffer 和同步状态机；
- `metal-api-vulkan`：AIR reflection、descriptor、exact `dispatchThreads`、
  Vulkan submit/readback、设备限制和超时保护；
- raw LLVM bitcode 与 offset-zero BitcodeWrapper 输入；普通多函数 `MTLB` typed-refuse；
- `metal-api-reims-vulkan`：离线 A/B executor，复用 reims 持久 Vulkan engine；
- `copy_word` 与 `indexed_boundary_dispatch`：文本、raw AIR、wrapped AIR、四 region
  barrier/index case；
- Linux/Lavapipe 和 Windows/RTX 5060 两套 executor 均通过上述 smoke。

当前本地提交：

- facade baseline：`252ca02656bcf80f943ea0bff7d924595ef919c0`；
- binary AIR：`4583d3deef7e34151e63fde88a00416b0d80926e`；
- reims engine seam (old `747580f` baseline): `bd62f89c3a6febdc7eb1d962d9f5e29fcb945305`；
- reims engine seam adapted to upstream `69a57dd`: local
  `3f19c66c7af392d4b588430a07119142c5cea8bd`；
- facade adapter pointing at that worktree: `metal-api-emulator@9c934cbf8a6a58724ca73bf4582ab6596c676349`。

关键限制：当前 `ComputeExecutor` 是“一次 submission snapshot → BufferUpdate”的
离线测试接口，不是可直接替换 native `compute_core` 的低层 provider。它目前还：

- 拒绝 native Metal 允许的同一 backing buffer 多 binding alias；
- 不表达 `dispatchThreadgroups`、stage-in、texture/sampler/session/ICB 全部合同；
- 不拥有 reims 的 MTLB function-object 解析；
- 不接 canonical Metal rail 的生产调用。

因此不能把当前 adapter 描述为“已经统一 backend”，只能描述为离线 provider 原型。

## 7. Native Metal seam inventory（Phase A 产物）

真正的 API 接缝不只在一个 compute 文件。第一批盘点覆盖整个 crate 的 direct Metal
调用点，按以下 owner 分类：

### 7.1 Neutral orchestration，暂不迁移

- `runtime/compute_exec/mod.rs`：`ComputeAccum`、bind/resolve、staging、dispatch
  dimensions、`ComputeStatus` 和 guest writeback orchestration；
- `runtime/compute_exec/metal.rs`：`execute_dispatch_metal` 负责 guest MTLB load、
  guest memory staging、ABI record 组装和 writeback；nested session 分支保留 Metal
  handle lifetime；
- `runtime/compute_session/metal.rs`：open encoder、session finish、nested job 和
  commit ordering；
- `reims-vgpu-core` / protocol / memory：继续保持现有 neutral ownership。

### 7.2 首批 provider seam：compute buffer

`backend/metal/compute.rs` 是实际 Metal API 集中处：

- `new_compute_pipeline_state`：function → compute PSO/cache；
- `compute_encode_on_encoder`：set pipeline、buffer/image/sampler/threadgroup binds、
  dispatch、返回 retain handles；
- `compute_core`：system device → thread-local queue → command buffer → compute encoder
  → end/commit/wait/status；
- `compute_writeback_from_mtl`：GPU 完成后的 buffer/image readback；
- `bind_compute_buffers`：binding validation、backing alias identity、no-copy/copy
  allocation、offset/attribute stride。

相邻 owner：

- `backend/metal/function.rs`：MTLB function load；
- `backend/metal/runtime.rs`：process-global device、thread-local queue、host buffer
  allocation；
- `backend/metal/raw_metal.rs`：nil-safe allocation、command/encoder/raw selector；
- `backend/metal/cache.rs`：function/PSO/reflection caches，和 render 路径共享。

### 7.3 暂不迁移的 direct-Metal consumers

- `runtime/icb/metal.rs`：ICB materialize/fill/cache；
- `runtime/draw/metal/{mod,icb}.rs`：render encoder、PSO、attachments；
- `backend/metal/resident.rs`：resident textures / `get_bytes`；
- `backend/metal/mipmap.rs`：blit command buffer；
- `backend/metal/window.rs`：CAMetalLayer/present；
- `runtime/scanout/metal.rs`：resident read；
- draw/blit/mapping/present 相关 Metal 调用点。

### 7.4 不能假设的生命周期合同

- Metal device 是 process-global `OnceCell`，queue 是 thread-local；不能随意跨线程传
  native handles；
- no-copy buffer 的 backing 地址必须稳定到 GPU completion；
- nested encoder 的 buffers/textures/PSO 必须由 session retain 到 commit；
- native Metal 用 `(backing_ptr, backing_len)` 复用 alias，并以首项写回；当前 facade
  的 alias refusal 只是 MVP 限制，不能当 native 合同。

## 8. Provider contract 的下一份交付物

在写 trait 前，先产出一份可评审的 contract，至少包含：

1. 对象/句柄：Device、Queue、Function、Pipeline、Buffer、CommandBuffer、Encoder；
2. 线程模型：哪些对象 thread-bound，哪些可 `Send + Sync`；
3. 生命周期：retain 规则、no-copy 地址规则、nested session 规则；
4. 内存可见性：upload、GPU completion、readback、guest writeback 顺序；
5. API 能力/refusal：dispatch 类型、alias、stage-in、texture/sampler 等逐项状态；
6. 错误模型：stable class/slug/fields 到 `ComputeStatus::RailRefused` 的映射；
7. trace schema：native Metal 与 Vulkan provider 使用同一输入 trace；
8. oracle：native Metal 输出/完成/错误作为 reference，Vulkan 只声明已覆盖集合。

第一批 contract 只覆盖 buffer-compute，但必须保留 alias 和 completion 的真实
语义，不能把当前 `ComputeExecutor` 快照接口直接冒充最终 trait。

## 9. 三重验证门

### Gate 1：provider parity

同一 trace 在 native Metal 与 Vulkan provider 上比较可观察语义。Linux/Lavapipe 是
开发控制环境；Windows/RTX 是目标 provider 环境；native Metal oracle 通常在 Apple
host 采集后回放/比较，不要求同一台机器同时运行两者。

### Gate 2：canonical rail integration

canonical Metal rail 实际调用 provider，neutral orchestration、session、writeback
和错误路径不再旁路 provider。只有这一步完成，才能说某个 API 类别接入 reims。

### Gate 3：VM E2E

VM 验证 guest Metal.framework/AppleParavirtGPU、wire、WHPX/KVM、guest RAM、dirty
tracking、display/present 的端到端行为。任何一门未过，都不能宣称整体 100% conformance。

## 10. 上库/PR 计划（当前不执行）

### 10.1 `hackintosh-research`

- 目标：提交本文、证据索引和路线说明；
- 不包含：源码二进制、VM 镜像、MTLB/AIR/SPIR-V 产物、日志、凭据；
- 先在本地 commit，用户确认后再 push 到用户的 research fork。

### 10.2 `metal-api-emulator`

- 目标：独立仓库，承载 provider 原型、offline parity runner 和 synthetic fixtures；
- 不向 `metal2vulkan` 提交 API facade；
- 先保留当前本地 `main` / `binary-air` 提交，确认仓库归属、可见性和 README 后再
  创建远端或 push；
- 后续 provider trait 成熟后，单独提交一个 focused PR/commit，不把 reims 全部代码
  镜像进来。

### 10.3 `reims-vgpu`

- 当前 `bd62f89` 只应作为本地 engine seam 试验，不应直接作为“统一 rail” PR；
- 下一次 PR 应包含：provider contract、native Metal 接线、Vulkan provider 接线、
  同一 trace parity 和逐类生产接线；
- PR 不应触碰 QEMU/WHPX/display 变化，也不应删除未迁移的 Vulkan path；
- 目标 fork/上游和提交拆分，等用户确认后再决定。

### 10.4 `metal2vulkan`

- 当前没有 API facade PR；
- 只有当 shader translator 本身需要结构性 bug fix，且有独立 regression test 时，
  才向该仓库提交 translator-only 变更。

## 11. 防偏航检查表

每次新增代码、测试或提交前回答：

1. 代码是在 neutral model、canonical Metal semantic path、provider，还是 shader
   translator 层？
2. 是否重新解释了 vGPU packet，造成第二套业务语义？
3. native Metal 与 Vulkan provider 是否能接收同一条 trace？
4. 生命周期、完成和错误是否由明确 owner 负责，未知能力是否 typed-refuse？
5. 测试是否能脱离 VM 快速运行，同时保留最终 VM 验收路径？
6. 结论范围是否严格等于实际覆盖的 API/资源/设备？
7. 是否意外把 MTLB/AIR/SPIR-V、镜像、日志或凭据加入仓库？
8. 是否需要 GitHub push/PR？若需要，先向用户报告并获得明确确认。

## 12. 决策记录

- **D1（local / proposed）：** Metal rail 是 host API semantic reference；Windows
  Vulkan 是 provider，不是第二套长期 vGPU 语义模型。
- **D2（现行约束）：** neutral protocol/core/memory/runtime owners 不移动；当前
  AGENTS.md 双 rail 实现契约继续有效，迁移是逐类 overlay。
- **D3（local / verified prototype）：** 当前 facade 已证明 compute buffer 的离线
  A/B 方法；它已在 `69a57dd` 的新 device-owned cache owner 上重放，但尚未接入
  canonical Metal rail。
- **D3a（verification boundary）：** 最新上游-69 lib clippy 仍有两项既存 upstream
  诊断（`handle_render_draw` 参数过多、测试 helper 冗余闭包）；它们不改变本原型的
  运行结果，也不应被写成“上游全量 lint 通过”。
- **D4（local / proposed）：** provider trait 先覆盖真实 native compute seam，再让
  `metal-api-core` 的高层 snapshot API 退居测试适配层。
- **D5（verified limitation）：** 普通 MTLB 暂缓；在函数名表/single-function
  contract 未正式建模前只能拒绝。
- **D6（hard release rule）：** 本文、源码和证据先留在本地；任何 GitHub 仓库、push、
  Issue 或 PR 动作必须先得到用户确认。

## 13. 关联资料

- [现有路线图](03-开发路线图.md)
- [本机实施计划](05-本机实施计划.md)
- `metal-api-emulator` README：本地 sibling workspace，尚未发布为本研究仓库文件
- [metal2vulkan 上游仓库](https://github.com/steelbrain/metal2vulkan)
- [用户提供的 Discord 原文](#2-discord-原意)
