# 在 Windows 上实现「新黑苹果」：可行性分析

## 0. TL;DR

- **可行，但按路径递减**：A（Windows x86-64 宿主 + x86_64 macOS 客户机，WHPX + 原生
  Vulkan）最现实，可奔着"日用"去；B（Windows x86-64 宿主 + arm64 macOS 客户机，
  TCG 软件翻译，27on86 移植）可作 headless 开发里程碑；C（Windows ARM64 宿主 +
  arm64 macOS 客户机）短期 blocked。
- **好消息**：reims-vgpu 的核心设计已经为 Windows 预留了位置——host-pointer 导入原语
  官方写明覆盖 Windows（`VK_EXT_external_memory_host`）、拷贝 rails 不依赖 dma-buf、
  窗口栈是跨平台的 winit、Rust 产品代码无 Linux 专属 cfg。
- **坏消息**：没有任何 Windows 路径被验证过；要补几个明确的小缺口（见 2.3），并亲自
  验证几个未知数（WHPX+macOS 兼容性、Windows ICD 的扩展支持）。

## 1. 所有路径共用的先决条件

| 组件 | Windows 侧方案 | 状态 |
|---|---|---|
| QEMU（qemu-reims-vgpu 分支） | MSYS2/mingw64 编译（QEMU 官方支持 Windows 构建） | 需动手构建 |
| Rust 产品库 | `x86_64-pc-windows-gnu` 目标产出 staticlib（mingw 归档，可直接被 gcc 链接；避免 MSVC/mingw ABI 混用问题）；或 msvc + llvm-lib 转换 | 需动手构建 |
| Vulkan | 原生 Windows Vulkan SDK；NVIDIA/AMD/Intel 驱动。ash 运行时加载 vulkan-1.dll，无静态导入问题 | 现成 |
| 外部工具 | LLVM 的 llvm-dis、SPIRV-Tools 的 spirv-val（官方 Windows 二进制） | 现成 |
| 加速器 | WHPX（需在 Windows 功能里启用「Windows Hypervisor Platform」）或 TCG | 现成 |
| 客户机镜像 | 路径 A：Linux 上 OSX-KVM 预置后导入（或 VMware 磁盘转换）；路径 B：必须有一台 Apple Silicon Mac 用 macosvm 预置 vmapple bundle + 提取 AVPBooter 固件 | 需外部设备 |

## 2. 路径 A：x86-64 Windows 宿主 + x86_64 macOS 13 客户机（最现实）

对应 reims-vgpu 的第一条通路（Linux KVM 版）的 Windows 变体：QEMU pc 机型 +
`reims-vgpu-pci` + OpenCore/OVMF + WHPX 加速 + 原生 Windows Vulkan 渲染。

### 2.1 为什么最现实

- x86_64 macOS 客户机路线成熟（OSX-KVM 生态十年积累：OpenCore、OVMF 分辨率模板等直接
  复用）；
- WHPX 在 x86-64 Windows 上从 Win10 2004 起就受 QEMU 支持（官方文档确认）；
- Windows 的 Vulkan 生态是第一等的；
- reims 的 x86 通路本身就是围绕"普通 PC 客户机"设计的（PCI 设备、页大小 12）。

### 2.2 设计上已就绪的部分

- Rust Vulkan 引擎、金属管线翻译、wire 解码：平台中立；
- 拷贝 rails：无 dma-buf 时自动启用（llvmpipe 宿主已证明此路径跑通）；
- host-window：winit 在 Windows 原生。

### 2.3 缺口清单（按工作量排序）

| # | 缺口 | 位置 | 工作量 | 影响 |
|---|---|---|---|---|
| 1 | meson 的 Rust 系统链接库是 Unix 专属（-lm/-ldl/-lpthread） | qemu fork `hw/display/meson.build` | 小（加 host_os == 'windows' 臂） | 不修无法链接 |
| 2 | surface 扩展选择没有 Windows 臂 | `crates/reims-vgpu/src/backend/vulkan/engine/context.rs` 690-720 行 | 小（加 `ash::khr::win32_surface::NAME`） | host-window 在 Windows 被 typed decline 拒绝 |
| 3 | 打包视图分配器无 Windows 臂 | `crates/reims-vgpu/src/runtime/host.rs` 的 alloc_block/bounce | 中（VirtualAlloc 或普通堆 + 拷贝填充） | 散页资源走 gather/copy，性能降级 |
| 4 | PCI 壳的 mmap 打包视图是 Linux 实现 | qemu fork `hw/display/reims-vgpu-pci.c` 248-285 行 | 中（Windows 上可先禁用、Rust 侧兜底） | 同上 |
| 5 | memfd 不存在 | x86 RAM 的共享 memfd 打包视图 | 接受降级（AGENTS.md 已预期：散页打包视图失败 → multi-run gather/copy） | 性能 |
| 6 | 脏页 witness 依赖 accelerator dirty-log | `reims-vgpu-dirty.c` | 验证（i386 WHPX 有 dirty tracking；arm64 WHPX 无完整实现） | x86 路径预期可用，需实测 |
| 7 | WHPX + macOS 客户机兼容性 | QEMU whpx 后端 | 未知数：WHPX 有已知 quirks（MMIO 上的 MMX/SSE 指令未实现、Win10 PIC 中断问题等），macOS 客户机在 WHPX 上没有 KVM 那样广泛的验证记录 | 若不可用退 TCG（慢）或嵌套 KVM |
| 8 | Windows 启动脚本 | vm/boot-x86.sh 的 PowerShell/批处理等价物 | 小-中 | 操作便利性 |

### 2.4 验证协议（沿用 AGENTS.md）

fail log 双通道读数、boot 与 panic/freeze 区分、n≥3 确认、`REIMS_VGPU_GUEST_IMPORT=off`
对照 boot、像素级对照（截图）、present_hz/offered_hz 成对读。

## 3. 路径 B：x86-64 Windows 宿主 + arm64 macOS 13 客户机（TCG 移植）

即把 27on86 项目搬到 Windows：QEMU vmapple 机型 + TCG 软件翻译。

- 27on86 在 Linux x86-64 已证明：macOS 13 可重复引导、8 vCPU、SSH、重启/关机干净。
  QEMU TCG 本身跨平台，**同一套代码在 Windows 上预期同样能引导**；
- 需要拆 vmapple.c 的 HVF 假设（`#include "system/hvf.h"` 等）。参考
  qemu-reims-vgpu 的 `steelbrain/macos-arm64-fixes` 分支（含大量 TCG 翻译器修正）；
- **前置条件**：AVPBooter 固件（从真 Mac 的 Virtualization.framework 里复制）+
  vmapple 客户机 bundle（macosvm 在真 Mac 上预置）。Windows 上无法凭空生成——**这条
  路必须有一台 Apple Silicon Mac 做预置**；
- 性能预期：TCG 下 headless 可用（SSH），GUI（挂 reims-vgpu-mmio）会非常慢。
  定位是开发/验证环境，不是日用。

## 4. 路径 C：Windows ARM64 宿主（Snapdragon X 等）+ arm64 macOS 客户机

- WHPX arm64：QEMU 官方文档确认支持，但要求 **Win11 24H2 + 2025 年 4 月可选更新**；
- **核心阻塞**：vmapple 需要 Apple 私有的指针认证 VM key 状态（Asahi 实验的内核补丁
  证明这套状态不属于标准 ARM 虚拟化上下文），WHPX API 不会暴露 → 短期不可行；
  除非未来 Microsoft 扩展 WHPX 或有人找到其他注入点；
- TCG 退路 = 路径 B 的慢速版（同架构也不能用 WHPX 加速，因为 vmapple 本身不兼容）；
- GPU 侧：Snapdragon X 的 WoA Vulkan 驱动存在但成熟度存疑（xemu 上游有 Snapdragon X
  系列渲染崩溃 issue）。长期研究目标。

## 5. 路径 D：不用跑 macOS 的开发（Windows 上完全可做、立刻可做）

- **metal2vulkan**：纯翻译器。llvm-dis + spirv-val 有 Windows 官方二进制；测试语料是
  自创的合成用例（仓库不携带第三方着色器）。`cargo test` / `cargo clippy` 全绿即可
  提交。
- **reims-vgpu-wire / decode**：解码器用 Apple 序列化器产出的 fixture 做测试，不需要
  虚拟机。
- 也就是说：**fork 的很大一部分贡献（解码、翻译、模型）完全可以在 Windows 上进行**，
  CI/真机验证放 Linux 或 Apple Silicon 宿主。

## 6. 风险与合规

- macOS EULA 对虚拟化的限制（通常要求 Apple 硬件）；AVPBooter 固件与恢复镜像必须合法
  获取；研究代码不传播任何受保护材料（上游仓库规则原文照搬）；
- 上游处于 alpha，C ABI、线格式、SPIR-V 输出都会变——保持跟版本、缓存产物要带版本；
- TCG 性能、WHPX quirks、Windows ICD 的 `VK_EXT_external_memory_host` 实际支持都要
  在目标机器上实测（vk_caps 行会点名缺什么）。

## 7. 推荐顺序

1. **先做路径 D**（零风险、立即上手、贡献可上游化）；
2. **再做路径 A**（Windows 上唯一有"日用"前景的组合；缺口明确、工作量可控）；
3. **路径 B 作为 headless 开发里程碑**（需要一台 Apple Silicon Mac 预置客户机）；
4. 路径 C 观望（等 WHPX 演进或社区突破）。


## 8. 战略补充：两条时间线要分开看

- **arm64 macOS 不是「macOS 27 及以后」**：Apple Silicon 的 macOS 从 macOS 11 Big Sur
  （2020）就开始了；Intel 线在收尾——macOS 26（Tahoe）是最后支持 Intel 的版本，
  macOS 27 起只有 arm64。arm64 macOS 客户机**今天就在跑**（27on86 的 macOS 13 引导、
  Asahi 实验的图形化 Setup Assistant 都是 arm64 客户机）。
- **x86 客户机路线有硬天花板**：x86 macOS 只能到 macOS 26 封顶，之后不存在新的 x86
  版本。所以路径 A 是「过渡期日用路线」，长期终局必然落在 arm64 macOS 客户机上：
  Apple Silicon 宿主（HVF/Asahi KVM）可近原生；x86 宿主只有 TCG（慢，研究定位）；
  Windows ARM 宿主卡在 WHPX 的 Apple PAuth 状态。
- 27on86 项目的名字即此意：用 TCG 在 x86 宿主上跑 arm64 的 macOS 27——因为未来的
  macOS 只存在于 arm64。


## 9. 宿主 GPU 与驱动选择：为什么 NVIDIA 重新变得有价值

- **旧黑苹果时代 NVIDIA 是死路**：High Sierra 之后无官方 Web 驱动，黑苹果社区被锁死在
  AMD GPU 上。新架构翻转了这一点——**渲染发生在宿主机，客户机只发命令流**，所以宿主
  GPU 的 Vulkan 驱动质量决定一切，NVIDIA 重新成为第一梯队选择。
- reims 的 Vulkan 后端对驱动的要求只有：Vulkan 1.2 基线、`VK_EXT_external_memory_host`
  （host-pointer 导入）、surface 呈现。**NVIDIA 的 Linux 专有驱动已经全部实测支持**——
  上游就是在 NVIDIA Linux 离散卡宿主上完成测量的（AGENTS.md 里 nvidia-smi 采样、SM/显存
  时钟记录、以及「amdgpu 与 NVIDIA 驱动在导入时调 get_user_pages」的观察都来自它）；
  NVIDIA Windows 驱动的 Vulkan 实现同样支持（移植后需用 vk_caps/vulkaninfo 实测确认）。
- **风险分层**：Windows 路径的成败决定因素不是 GPU 驱动层（NVIDIA Windows 驱动确实是
  三个平台里最成熟的），而是**虚拟化层**——WHPX 跑 macOS 客户机是最大的未知数
  （缺口 #7），而 Linux KVM 是 OSX-KVM 十年生态验证过的。所以「Windows 更好」在 GPU 层
  成立，在 hypervisor 层反而 Linux 是已验证的标准答案。
- **实用主义双轨建议**：Linux + NVIDIA 先跑通 x86 通路拿到实测基线（近原生的数字就来自
  它），Windows 移植的每一步都与基线对照 boot（符合 AGENTS.md 的验证文化）；客户机镜像
  经 rails import/export 在两轨间复用。


## 10. Windows 上没有 KVM：加速器层次澄清

- **KVM 是 Linux 内核子系统**，不可能原生运行在 Windows 上。Windows 上占据同一位置的
  第一层接口是 **WHPX（Windows Hypervisor Platform）**——Hyper-V hypervisor 的用户态
  API。QEMU 的 `-accel whpx` 与 Linux 的 `-accel kvm` 是**同一角色的两个实现**：
  同样是一层虚拟化、同样硬件加速，CPU 侧性能预期同样是 ~95%+ 量级，不存在"WHPX 多一
  层"这回事。
- 真正"多一层"的是：WSL2 里跑 KVM（WSL2 本身是 Hyper-V 里的 VM，内部再虚拟 = 嵌套两层。
  x86 上 WSL2 的 nestedVirtualization 疑似已能提供 /dev/kvm——microsoft/WSL #13262 /
  #13796，但 GPU 还要过 /dev/dxg 半虚拟层，`VK_EXT_external_memory_host` 大概率缺失，
  只能走拷贝 rails）；Hyper-V 里的完整 Linux VM + 嵌套 KVM 同理。都不建议作为主路线。
- 所以 WHPX 的风险**不在层数或架构，而在实现成熟度**：QEMU whpx 后端有已知 quirks 清单
  （MMIO 上的 MMX/SSE 指令未实现、Win10 PIC 中断问题等），macOS 客户机在 WHPX 上没有
  KVM 那样十年的验证记录。它是"未验证路线"，不是"注定更慢的路线"。
- **决策含义**：不想折腾 Linux 时，正确顺序是 路径 D → 阶段 1/2 构建 → 直接 WHPX 冒烟；
  Linux 基线只在 WHPX 被证伪后才必要，且届时只需最小安装（装驱动 + 跑 boot 脚本做
  对照），不是完整开发环境。



