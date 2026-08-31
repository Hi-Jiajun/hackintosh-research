# 第十四轮：WHPX 通路进入桌面（2026-08-31）

## 里程碑

- 工具链补齐（mingw-w64 llvm-tools + spirv-tools：llvm-dis.exe、
  spirv-val.exe）后，metal2vulkan 的 shader 翻译正常工作：
  draws_fail 从 56592 → 0，present_black 从 578 → 0。
- guest 完成 boot：WindowServer + loginwindow 运行，AppleKeyStore
  unlock successful，用户确认进入桌面（可交互）。
- 上一轮黑屏根因：llvm-dis 缺失 → m2v_vertex_translate 拒绝 →
  draws_skipped_after_engine_refusal ×32655。

## 遗留问题 1：boot 极慢（~15-20 分钟）

- WHPX 特有：内核早期 rep movsb #PF 重试循环（详见 round12/13），
  Hyper-V 直接 deliver 后续 #PF（不 exit），长时间 IF=0 慢速推进。
- KVM 对照无此阶段。定性：性能问题，后续优化方向：
  QEMU 侧 PF 处理 / Hyper-V 页表刷新路径调查。

## 遗留问题 2：桌面闪烁

- fail log：qemu_map_pages_callback_failed rc=-1 ×552（另有
  air_loading ×257 为翻译排队，正常）。
- C shim reims_vgpu_pci_map_pages 的 _WIN32 分支直接 return -1
  （无 mmap）；失败 GPA 含 0x45b954000（≈18.7GB，超出 16GB RAM，
  位于 fb 区 0x400400000 之外）→ 设备映射区外的 GPA 请求。
- 机制：guest 显存 backing 页无法打包别名导出 → copying rails
  部分失败 → 帧纹理缺失 → 闪烁。
- 修复方向（下一步）：
  a) Windows 下 memory-backend-file + share=on（CreateFileMapping）
     给 C shim 提供可映射的 guest RAM；
  b) 梳理 guest GVA backing 的 GPA 区间合同（为何请求 18.7GB），
     按 AGENTS.md 流程恢复合同后再改。

