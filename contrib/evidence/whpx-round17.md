# 第十七轮：闪烁定位到 copying rail（2026-08-31）

## 诊断 boot 证据（Windows 树 C 侧诊断）

- reims-map-fail ×134：全部 seq=0（fragmented 列表）；fastfail=0——
  C 侧快速路径健康，无 packed 列表失败。
- 典型失败：type4 surface n=2040（1920x1080 BGRA，8.3MB），
  physical_runs=493（极端碎片化）。
- Rust 侧链路：contig_view_refused（typed refusal）→ type11_load_seed
  outcome=guest_pages → vk_memory_type_pick DeviceLocal 8.8MB 成功。
- 结论：map 失败是设计内拒绝；闪烁/条纹 = copying rail 缺陷
  （Windows 通路唯一路径；Linux 有 mmap alias 不走此 rail，故 KVM 无闪烁）。

## 下一步

- 深挖 type11_load_seed/guest_pages 填充路径：条纹疑似 row-pitch/
  布局错误，闪烁疑似拷贝-呈现同步问题。
