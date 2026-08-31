# 第八轮：KVM 对照与 kernel 形态差异（2026-08-31）

## KVM 对照 VM 状态

- 正常 boot 到桌面（first guest frame presented）。
- SSH（macos-vm 别名）可用；sysctl kern.slide 返回 1（已弃用，无真实值）。
- 物理搜索定位 kernel 头：**0x1e200000**（482MB）。

## 关键发现：两个 guest 的 kernel 形态不同

- KVM：**prelinked kernelcache**（ncmds=870、__REGION0-211、
  __TEXT vmaddr=0xffffff801e200000——slide 已并入 vmaddr）。
- WHPX：standalone kernel（ncmds=3）。
- 这是 OpenCore 配置差异（KVM 用 rails 快照的配置）。

## KVM 的 cpu_data 字段对照未完成

KVM kernelcache 无符号表（release），cpu_data 在 heap（地址未知），
无法定位 cpu_data_ptr 数组。

## 结论

WHPX 的除零（cpu_data 拓扑字段 1/16）与 KVM 的对照仍缺最后一环；
下一步候选：
1. KVM 换 standalone kernel 重试（或 WHPX 换 prelinkedkernel）；
2. WHPX 侧找 cpu_data 字段（+0xc8/+0xcc）的写者（cpu discovery 代码），
   定位 WHPX 与 KVM 的输入差异。