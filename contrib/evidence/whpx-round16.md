# 第十六轮：闪烁根因——scattered 页在 Windows 无打包视图（2026-08-31）

## KVM 对照（关键证据）

- KVM（WSL）侧 qemu_map_pages_callback_failed = **0**；WHPX（Windows）
  侧 552 次。两侧 QEMU RAMBlock 布局完全一致（16GB RAM + fb @
  0x400400000, 16MB）→ 差异不在 QEMU 布局。
- WHPX 失败 GPA 分布修正：0x2x ×1117、0x3x ×129、0x4x ×31——
  **大部分在 RAM 内**（前一轮 head -20 采样偏差误判为全在
  aperture 区）。
- 失败请求 page_count=2/3/4 的小页组也失败 → 非 plen 截断；
  GPA 是 scattered（guest 物理页不连续）。

## 根因（合同级）

- reims 需要把 guest 的 scattered 物理页打包成连续 host-VA 视图
  （GVA backing 导出）。
- Linux：memfd share + MAP_SHARED 别名视图 → 成功。
- Windows：无 mmap；C shim _WIN32 分支直接 return -1；copying
  rails 兜底不完整 → 部分帧纹理缺失 → 闪烁。

## 修复方向

- A（结构正路）：Windows 下 memory-backend-file + share=on
  （CreateFileMapping）+ C shim _WIN32 分支用 MapViewOfFile
  实现打包别名视图（对齐 Linux 能力）。
- B：修 copying rails（map 失败时 QEMU HostOps 逐页拷贝兜底）
  的完整性。
- 两者都需按 AGENTS.md 合同流程（先 checkpoint，后实现）。

