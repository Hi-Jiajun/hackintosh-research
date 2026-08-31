# 第二十轮：A/B 回退遇构建问题 + 状态固化（2026-09-01）

## A/B 回退状态

- 目标：验证上游同步（Rust c7c4183 + C 1c9c3aa3）是否引入 panic。
- 回退组合（Windows 树原始状态）：Rust = 2835b6c0（PR #57），
  C = refs/tmp/pr2（3aba725a，无 map_pages failure 参数）。
- 构建结果：Rust 2m52s 完成、link 成功，但 QEMU 启动报
  'reims-vgpu-pci' is not a valid device model name——
  hw_display_reims-vgpu-pci.c.obj 仅 721 字节（stub），设备未注册。
- 待查：hw/display/meson.build 的 shim 编译条件（为何回退后变 stub；
  上游同步版与原始版的 meson 差异）。

## 上游同步版 panic 证据（已入库 round19）

- 两次 boot 均 kext 列表加载完成后 panic（stackshot + corefile）。

## 资料

- DeepWiki（deepwiki.com/steelbrain/reims-vgpu）为 SPA，
  内容 JS 加载，需抓 _next/data JSON 或直接网页阅读。

