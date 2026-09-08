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
- provider contract B0 scaffold：`metal-api-emulator@b4dbb21`、capability admission
  `@9d0ac29`（本地，尚未发布）。
- Vulkan reflection/limits mapping：`metal-api-emulator@be9c04e`（本地，尚未发布；
  不改变现有 executor 执行路径）。
- Vulkan provider mapping fixtures：`metal-api-emulator@1073067`（本地，尚未发布）。

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

## 8. Provider contract 交付物

首版契约已整理为：[Metal provider contract v0](11-Metal-provider-contract-v0.md)。
它先作为设计基线，纯值 scaffold 已在本地实现；不改变现有 facade 或 reims 生产路径。
下一步仍是根据评审结果稳定 trait 的具体对象边界。

契约覆盖的最小内容包括：

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
- **D6（hard release rule）：** research 文档已在用户确认后发布；emulator/reims 源码、
  新仓库、Issue 或 PR 动作仍必须先得到用户确认。

## 13. 当前进度快照（2026-09-08）

> 本节是状态记录，不改变第 1–12 节的目标与合同。结论以当前仓库、可复现测试和已归档证据为准。

### 13.1 暂定目标（复述）

不替换 guest `Metal.framework`，不做任意 macOS Mach-O 在 Windows 上的二进制兼容；目标是在 reims 宿主侧建立统一的 Metal API provider：

```text
guest Metal.framework / AppleParavirtGPU / vGPU wire
  -> reims neutral decode / transaction / memory owners
  -> canonical Metal semantic execution path
  -> provider: macOS native Metal | Windows Vulkan-backed emulator
```

让同一份调用轨迹可在 native Metal 与 Windows Vulkan provider 上离线复现；`metal2vulkan` 仍只是 shader translator；neutral owners 不迁移；direct Vulkan rail 在迁移期保留为控制路径；逐类 parity 后才切换。

### 13.2 已完成

- `metal-api-emulator` 的 `shared-provider-objects` 分支（推送目标 `origin/main`）：HEAD `8d2a4ca`（2026-09-08），已推送；其中 `69dab48` 为 metal2vulkan pin 升级，`76f6102` 为异步 completion/readback 契约，`31a0ba8` 为 Vulkan 真实异步 provider，`ebca75e` 为 native Metal completion handler 与共享 `CompletionRecord`，`15bf79a` 为 Vulkan 设备侧 fence 等待，`45a7427` 为共享可配置 observation deadline，`6e5df5b` 为 reims 适配器跨 translator 版本桥接，`81d6cbe` 把可选 reims 集成纳入 CI，`eed4a53` 为 Vulkan 超时提交的 retirement 回收，`909a5fb` 为显式取消契约与对象 API，`0630022` 为 native 超时观察后保持可用，`008fe7d` 把 README 与 provider 文档同步到新的超时/取消语义，`3bf3c04` 新增 v8 binary AIR encoding conformance 套件（raw/wrapped），`d8e42bf` 让 Swift native oracle 接受 v8，`042acc8` 把 v8 套件写入 README 与 provider/conformance 文档并扩展 object-path Python 测试，`a0b10e1` 增加有界放弃预算、`ProviderHealth` 与设备丢失销毁路径，`7e67a0e` 增加 native `MTLCommandBufferError::DeviceRemoved` 分类与对象 API submit 期 `DeviceLost`/`Exhausted` 错误矩阵，`8d2a4ca` 把 Windows RTX 5060 v1–v8 验证写入 README 与 provider 文档。
- 同步 buffer-compute 子集：`Device / Library / Function / ComputePipelineState / CommandQueue / CommandBuffer / ComputeCommandEncoder / Buffer` 对象 API；文本 LLVM IR、raw AIR、offset-zero wrapper；exact-thread dispatch（含 tail）；有界资源校验；单命令缓冲区最多 8 个串行 pass；每 pass 资源子集；多 pipeline 切换；跨 pass 重绑定；最多 64 个稳定 owned view；整个 command buffer 单次提交；完整校验后写回；完整 readback。
- 共享 provider：`PipelineProvider` 统一编译、元数据和释放；native Rust Metal backend 接受 6 个审查过的 MSL fixture；Vulkan backend 接受文本 IR/raw AIR；已有 device epoch、结构化 refusal、static/affine footprint proof、completion token 等契约。
- binary AIR encoding conformance（`3bf3c04` + `d8e42bf`）：`suite-v8.json` 复用 v7 的三个 `subset_chain_*` 用例，新增 `air_encoding` 字段（两个 raw、一个 Apple wrapper）；Vulkan direct/object/async 三条 rail 用 `llvm-as` 汇编审查过的 `.ll` 后提交所选编码，Swift 与 Rust native rail 继续使用审查过的 MSL。`conformance/compare.py`、`test_suite_v8.py`、CI capture 循环与五路径 compare 均已接入；`d8e42bf` 修复了 Swift oracle 只接受到 v7 的缺口（首个 run `34223201314` 因此在 native-oracle-build 失败，standalone/reims 通过）。
- 验证（v8 云端）：CI run `34223294821` 四 job 全绿：standalone 中 151 Rust（core 96 / native 7 / Vulkan 34 / capture 14）+ 115 Python 通过，Lavapipe v1–v8 direct/object/async 共 29 cases 每 rail 通过；native-oracle-build 在 Apple Paravirtual device（macOS 15.7.9）捕获 v8 成功；compare-captures 五路 v1–v8（29 cases）一致；reims-integration 通过。证据归档在 `evidence/conformance-v8-d8e42bf-2026-09-08/run-34223294821/`。文档同步 `042acc8` 的 CI run `34223676120` 同样四 job 全绿，证据在 `evidence/provider-v8-docs-042acc8-2026-09-08/run-34223676120/`。
- 有界放弃与 health（`a0b10e1`）：新增 `ProviderHealth { Usable, DeviceLost, Exhausted }`、单调 `AbandonmentBudget`/`AbandonmentLedger`（Vulkan 默认 1 个 submission / 64 MiB，native 默认 8 个 / 64 MiB）；Vulkan retirement 线程区分 fence 完成、观察超时与设备丢失，`VK_ERROR_DEVICE_LOST` 走销毁路径而不再泄漏，provider 把 `DeviceLost` 映射为 `device_lost` + `RetryAfterRecreate`、把 `Exhausted` 映射为 `provider_unavailable` + `RetryAfterRecreate`；native 按 submission 记录 owned bytes，deadline 到期和释放仍 running 的 completion 计入预算，并暴露对称的 `with_abandonment_budget()`/`health()`/`abandonment_stats()`。设计见 [13-有界回收与错误传播设计](13-有界回收与错误传播设计.md)。
- 验证（`a0b10e1` 云端）：CI run `34224461611` 四 job 全绿：standalone 中 156 Rust（core 99 / native 7 / Vulkan 36 / capture 14）+ 115 Python 通过，Lavapipe v1–v8 direct/object/async 共 29 cases 每 rail 通过；native-oracle-build 在 Apple Paravirtual device 上运行带 health/预算断言的 deadline 测试并捕获 v8；compare-captures 五路 v1–v8 一致；reims-integration 通过。证据归档在 `evidence/provider-abandonment-a0b10e1-2026-09-08/run-34224461611/`。
- 验证（`7e67a0e` 云端）：CI run `34225380992` 四 job 全绿：standalone 中 159 Rust（core 100 / native 9 / Vulkan 36 / capture 14）+ 115 Python 通过，Lavapipe v1–v8 direct/object/async 共 29 cases 每 rail 通过；native-oracle-build 在 Apple Paravirtual device（macOS 15.7.9）执行 100 core + 10 native 测试（含 macOS-only 的 code 11 常量断言）并捕获 v8；compare-captures 五路 v1–v8 一致；reims-integration 通过。证据归档在 `evidence/device-removal-7e67a0e-2026-09-08/run-34225380992/`。
- Windows/RTX 5060 v1–v8：`7e67a0e` 的 Windows GNU debug 二进制在 NVIDIA GeForce RTX 5060 上通过 `provider-smoke`（12 项）和 v1–v8 direct/object/async-object 三条 rail（每 rail 29 cases），包含 v8 raw/wrapped；二进制 SHA-256 与证据在 `evidence/windows-rtx5060-v8-7e67a0e-2026-09-08/manifest.md`。
- 验证（`8d2a4ca` 云端）：CI run `34226091972` 四 job 全绿；该提交只改 README/PROVIDER-B1/PROVIDER-OBJECTS 三份文档，Rust/Python 源码与 `7e67a0e` 相同，159 Rust + 115 Python 与五路 v1–v8 结果不变。证据归档在 `evidence/windows-v8-docs-8d2a4ca-2026-09-08/run-34226091972/`。
- 异步 completion/readback 契约（`76f6102`）：新增 `CompletionReadback` 与 `ComputeProvider::readback`；provider 可返回 `Submitted`，对象 API 在首次 `wait_until_completed` 时重试非终态超时、按精确 trace 校验 readback、全部校验通过后写回，并在整个 commit→completion 窗口用 buffer reservation 阻止 CPU 访问和重叠命令；`Failed` / `DeviceLost` / `SubmittedUnknown` 不写回；丢弃 pending command 释放 reservation 与 completion 记录，但不声称 GPU 已退役。
- Vulkan 真实异步 provider（`31a0ba8`）：`VulkanComputeProvider::with_async_execution(true)` 在 `submit` 时准备 owned-byte 请求、返回 `Submitted`，由 worker 在共享 queue lock 下执行；`wait` 按调用方 timeout 观察共享 completion record，`readback` 返回规范 writeback。默认同步模式、direct trace rail 与五路径 capture 不变；`provider-capture --api objects --async` 在 commit 后断言 `Committed` + `Submitted`，确保走真实异步路径。
- 验证（`31a0ba8` 本地）：86 core / 7 native / 37 Vulkan / 14 capture = 144 Rust + 113 Python 全通过；Lavapipe standalone/provider smoke、v1–v7 同步 direct/object 捕获、v1–v7 异步 object 捕获（共 26 case）全部通过；fmt/clippy/rustdoc 通过。
- native Metal 异步 provider（`ebca75e`）：`NativeMetalProvider::with_async_execution(true)` 在 commit 后返回 `Submitted`，注册 `MTLCommandBuffer` completion handler 回填共享 `metal_api_core::completion::CompletionRecord`；handler 保留 device/queue/pipeline/buffer 直至执行完成，首个终态胜出，20 秒 deadline 报 `metal_completion_unknown` + `SubmittedUnknown` 并弃用 provider。默认同步模式不变；`provider-capture --api objects --async` 在 native backend 上同样断言 commit 后的 `Submitted` 并走 wait/readback。
- 验证（`ebca75e` 本地与云端）：90 core / 7 native / 34 Vulkan / 14 capture = 145 Rust + 113 Python 全通过；Lavapipe standalone/provider smoke 与 v1–v7 异步 object 捕获通过；macOS target check/clippy 通过；CI run `34219297172` 成功：standalone、native-oracle（含 v1–v7 native async object 捕获）、五路 compare（26 cases）全绿。证据归档在 `evidence/native-async-ebca75e-2026-09-08/run-34219297172/`。
- Vulkan 设备侧 fence 等待（`15bf79a`）：异步 `submit` 在调用线程完成 plan/record/`queue_submit` 并返回 `Submitted`，不再为每个提交开 worker；`wait` 在调用线程按 caller timeout（上限 20 秒 deadline）等待 per-submission fence 并 readback；`release_completion` 把仍 pending 的提交交给共享 retirement 线程，provider drop 也会 drain。20 秒 deadline 报 `vulkan-completion-unknown` + `SubmittedUnknown` 并弃用 executor。默认同步路径与 direct rail 不变。
- 验证（`15bf79a` 本地与云端）：145 Rust（90/7/34/14）+ 113 Python 全通过；Lavapipe standalone/provider smoke 与 v1–v7 direct/object/async-object 捕获全部通过；macOS target check/clippy 通过；CI run `34219887881` 成功：standalone、native-oracle（含 v1–v7 async object 捕获）、五路 compare（26 cases）全绿。证据归档在 `evidence/vulkan-fence-15bf79a-2026-09-08/run-34219887881/`。
- 共享 observation deadline（`45a7427`）：`metal_api_core::completion::ObservationDeadline` 统一 remaining/expired/clamp 语义；两端 provider 新增 `with_observation_deadline`，默认 20 秒；deadline 过期后返回各自的 terminal unknown completion 而不是非终态 `TimedOut`。两个新 core 测试覆盖 clamp 与 expiry。
- 验证（`45a7427` 本地与云端）：147 Rust（92/7/34/14）+ 113 Python 全通过；Lavapipe v1–v7 async-object 捕获通过；macOS target check/clippy 通过；CI run `34220240822` 成功：standalone、native-oracle、五路 compare（26 cases）全绿。证据归档在 `evidence/observation-deadline-45a7427-2026-09-08/run-34220240822/`。
- 可选 reims 适配器修复（`6e5df5b`）：pin 升级到 43c46ac 后，适配器不再把新版 `KernelDispatch` 传给 vendored reims 的 9e0e99a API，而是用当前 reflection 的 `validate`/`push_constant_range`/`plan` 直接构造 `ComputeDispatch::Regions`；两版 region 与 48-byte payload ABI 相同。`Cargo.lock` 记录两个 translator revision，README 说明 reims 需先升级才能共用同一类型。
- 验证（`6e5df5b` 本地）：`integration/reims` 3 个适配器单测通过；Lavapipe `reims-smoke` 的 `copy_word`、raw/wrapped binary AIR、`indexed_boundary_dispatch`（30 words / 4 regions）与 suite executor 全部 PASS。根 workspace CI run `34220576077` 成功（integration workspace 仍不在 CI 中）。证据归档在 `evidence/reims-bridge-6e5df5b-2026-09-08/`。
- 可选 reims 集成纳入 CI（`81d6cbe`）：新增 `reims-integration` job，运行 `prepare.py`（拉取 pinned 69a57dd + `compute-facade.patch`）、3 个适配器单测和 Lavapipe `reims-smoke`。CI run `34220849337` 四个 job 全绿。证据归档在 `evidence/reims-ci-81d6cbe-2026-09-08/run-34220849337/`。
- Vulkan 超时提交回收（`eed4a53`）：observation deadline 过期时不再原地 drop 正在执行的 `PendingExecution`（那会 poison 整个 executor），而是交给共享 retirement 线程等待 fence 后释放；只有 fence 始终不 signal 时才 poison/泄漏到进程退出。`provider-smoke` 新增 `run_timeout_reclamation`：零 deadline 得到 `vulkan-completion-unknown` + `SubmittedUnknown` 后，共享同一 executor 的第二个 provider 仍能执行并回读 `copy_word` golden。该回归测试在回退 `compute_provider.rs` 后确实以 `provider_unavailable` 失败。CI run `34221388956` 四个 job 全绿。证据归档在 `evidence/vulkan-timeout-reclaim-eed4a53-2026-09-08/run-34221388956/`。
- 显式取消契约（`909a5fb`）：新增 `CompletionDisposition::Cancelled`、`CompletionRecord::cancel`（first-terminal-wins）与 `ComputeProvider::cancel`；Vulkan 把 pending 交给 retirement 线程，native 依赖 completion handler 释放资源。对象 API 新增 `CommandBuffer::cancel`，释放 host reservation 并以 `CompletionUnavailable(Cancelled)` 报告后续观察；取消与完成竞态、provider 拒绝取消都不会写入未经校验的字节。`provider-smoke` 新增 `run_cancellation`，验证 slot 释放、`wait` 保持 `Cancelled`、`readback` 报 `completion_cancelled`、同一 provider 继续执行新工作。CI run `34221535473` 四个 job 全绿。证据归档在 `evidence/completion-cancel-909a5fb-2026-09-08/run-34221535473/`。
- native 超时观察后保持可用（`0630022`）：`fail_deadline` 不再设置 `async_abandoned`；completion handler 本就保留 device/queue/pipeline/buffer 直到 Metal 报告终态再释放，所以 deadline 过期只需发布 `SubmittedUnknown`，context 仍可用于后续提交。只有 handler 观察到 `MTLCommandBufferStatus::Error` 才禁用新工作。新增 macOS 设备回归测试 `deadline_expiry_keeps_the_native_provider_usable`（零 deadline 下第二次 submit 仍返回 `Submitted`），在 macOS runner 上实际执行并通过。CI run `34221811367` 四个 job 全绿。证据归档在 `evidence/native-deadline-recovery-0630022-2026-09-08/run-34221811367/`。
- 验证（`489b489`）：本地 133 Rust + 113 Python 测试通过；云端五路径（Swift native、Vulkan direct、Rust Metal provider、Vulkan object API、Rust Metal object API）在 v1–v7 共 26 case/path 上一致，CI `34011824447`；更早 RTX 5060/Lavapipe smoke 通过。证据在 `metal-api-emulator/evidence/`。

### 13.3 未完成

1. 异步提交与结果回收：契约、对象 API、Vulkan 与 native Metal provider 均已支持 `Submitted` → `wait` → `readback` → 写回（Vulkan 用设备侧 fence，native 用 `MTLCommandBuffer` handler）、共享可配置的 observation deadline（默认 20 秒）、显式取消（`CompletionDisposition::Cancelled` + `ComputeProvider::cancel` + `CommandBuffer::cancel`）以及超时后的资源回收（Vulkan retirement 线程 / native completion handler）；`a0b10e1` 已加入 provider 级 `AbandonmentBudget`、`ProviderHealth` 与 Vulkan 设备丢失销毁路径，预算耗尽后 fail closed；`7e67a0e` 补齐 native `MTLCommandBufferError::DeviceRemoved` 分类与对象 API 提交期 `DeviceLost`/`Exhausted` 错误矩阵。仍缺跨进程 completion 协议、completion 驱动的 lease 生命周期，以及更完整的生产错误传播。
2. 通用 shader 支持：只接受固定、审查过的 shader/source/entry/layout/footprint；v8 已覆盖 raw AIR 与 Apple wrapper 两种固定编码，但仍缺任意 AIR/MSL 编译、通用反射、地址计算、更多原子操作、纹理访问、动态资源索引、通用 MTLB 函数名解析和 Windows MSL 编译。
3. 真实 guest memory 生命周期：buffer 数据主要在 provider 边界内管理；缺 guest allocation、映射、脏页、上传/回读、失效、迁移，以及 GPU 执行期间 CPU 修改资源的语义；snapshot 不持有真实 guest page；同一 backing buffer 多 binding alias 仍被拒绝。
4. reims 生产接入：只有可选、离线的 Vulkan A/B executor（pin 升级后适配器已在 `6e5df5b` 修复，并由 `81d6cbe` 的 CI job 持续验证）；生产 guest/display 路径未调用 canonical Metal provider；缺跨进程协议、真实设备生命周期、生产队列调度和错误传播；Gate 2 未通过。
5. 图形与显示路径：纹理、render pass、sampler、render pipeline、presentation、swapchain、heaps、ICB 均未实现；当前范围仍是 compute buffer 子集。
6. 三重验证门：Gate 1 只在受限 fixture 内成立；Gate 2、Gate 3（VM E2E：guest Metal.framework/AppleParavirtGPU/wire/WHPX/KVM/guest RAM/dirty tracking/display）未完成，不能宣称 100% conformance。

### 13.4 外部依赖状态（2026-09-08）

- `metal2vulkan`：pin 已从 `9e0e99a` 升级到 upstream master `43c46ac`（2026-09-06，squash 680 commits，reflection v41→v55）。主工作区验证：133 Rust + 113 Python 测试通过，Lavapipe standalone/provider smoke 通过，clippy/fmt 通过。已提交 `69dab48` 并推送到 `main`；CI run `34215960535` 成功：standalone、macOS native-oracle、五路 compare 全绿，v1–v7 共 26 cases 的 native Metal / Vulkan / Rust Metal / Vulkan objects / Rust Metal objects 结果一致。证据归档在 `evidence/metal2vulkan-43c46ac-2026-09-08/run-34215960535/`。WSL 内 RTX 5060 Vulkan 枚举失败属环境限制。详见 [12-上游更新评估-2026-09-08](12-上游更新评估-2026-09-08.md)。
- `reims-vgpu`：facade 基线 `69a57dd` 与 upstream master 一致；新工作集中在 open PR #78/#79/#80，其中 #79（guest write release ordering）和 #80（synthesized read sampler reflection）与 provider/内存生命周期直接相关。
- `metal-api-emulator` 远端 `main` 已更新到 `8d2a4ca`（含 pin 升级、异步 completion/readback 契约、Vulkan 与 native Metal 异步 provider、Vulkan 设备侧 fence 等待、共享可配置 observation deadline、reims 适配器桥接与 reims CI job、Vulkan 超时回收、显式取消、native 超时后保持可用、v8 binary AIR encoding 套件、有界放弃预算与 health、native device removal 分类与对象 API 提交期错误矩阵）；本地与远端同步。`76f6102` 的 CI run `34216817216` 成功；`31a0ba8` 的 CI run `34218661059` 成功；`ebca75e` 的 CI run `34219297172` 成功；`15bf79a` 的 CI run `34219887881` 成功；`45a7427` 的 CI run `34220240822` 成功；`6e5df5b` 的 CI run `34220576077` 成功；`81d6cbe` 的 CI run `34220849337` 成功；`eed4a53` 的 CI run `34221388956` 成功；`909a5fb` 的 CI run `34221535473` 成功；`0630022` 的 CI run `34221811367` 成功；`008fe7d` 的 CI run `34222084119` 成功；`3bf3c04` 的 CI run `34223201314` 在 native-oracle-build 失败（oracle 拒绝 v8，standalone/reims 通过），由 `d8e42bf` 修复；`d8e42bf` 的 CI run `34223294821` 成功（四 job 全绿，五路 v1–v8 共 29 cases 一致）；`042acc8` 的 CI run `34223676120` 成功（四 job 全绿）；`a0b10e1` 的 CI run `34224461611` 成功（四 job 全绿，156 Rust + 115 Python，五路 v1–v8 一致）；`7e67a0e` 的 CI run `34225380992` 成功（四 job 全绿，159 Rust + 115 Python，五路 v1–v8 一致）；`8d2a4ca` 的 CI run `34226091972` 成功（四 job 全绿）。证据归档在 `evidence/conformance-v8-d8e42bf-2026-09-08/run-34223294821/`、`evidence/provider-v8-docs-042acc8-2026-09-08/run-34223676120/`、`evidence/provider-abandonment-a0b10e1-2026-09-08/run-34224461611/`、`evidence/device-removal-7e67a0e-2026-09-08/run-34225380992/` 与 `evidence/windows-v8-docs-8d2a4ca-2026-09-08/run-34226091972/`。
- `qemu-reims-vgpu`：master 仍为 `300438f`（2026-07-24），没有与 provider 直接相关的新 master 变化。

### 13.5 下一步

- 有界回收：`a0b10e1`/`7e67a0e` 已完成 provider 级预算、`ProviderHealth`、Vulkan 设备丢失销毁路径、native `MTLCommandBufferError::DeviceRemoved` 分类与对象 API 的 `DeviceLost`/`Exhausted` 端到端断言；下一步做跨进程 completion 与 completion 驱动的 lease 释放。
- v8 conformance 已完成：`3bf3c04`/`d8e42bf`/`042acc8` 增加 binary AIR encoding 维度（raw/wrapped），并保持五路 v1–v8（29 cases）全绿；下一步评估新的语义维度（候选：更复杂的 buffer 布局/offset、多 command buffer 提交或错误传播路径），仍不宣称完整 Metal conformance。
- Windows/RTX 5060 v8 已完成：`7e67a0e` 的二进制在 RTX 5060 上跑通 v1–v8 三条 rail（每 rail 29 cases），证据在 `evidence/windows-rtx5060-v8-7e67a0e-2026-09-08/`。
- 跟踪 reims PR #79/#80；合并后再决定 facade worktree 是否 rebase。
- 保持 direct Vulkan rail 作为控制路径，直到 Gate 1/2/3 逐类通过。

## 14. 关联资料

- [现有路线图](03-开发路线图.md)
- [本机实施计划](05-本机实施计划.md)
- [上游更新评估（2026-09-08）](12-上游更新评估-2026-09-08.md)
- `metal-api-emulator` README：本地 sibling workspace，尚未发布为本研究仓库文件
- [metal2vulkan 上游仓库](https://github.com/steelbrain/metal2vulkan)
- [用户提供的 Discord 原文](#2-discord-原意)
