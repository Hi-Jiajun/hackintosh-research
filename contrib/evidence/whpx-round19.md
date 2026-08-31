# 第十九轮：boot 慢数据收集 + 上游同步版 panic 复现（2026-09-01）

## boot 慢数据（第 4 次诊断 boot，slide 0x1A800000）

- PF exit 计数 = 1（"只 exit 一次"模式再次确认）。
- 30 轮密集采样（22 轮有效）：repmovsb ×3、idle ×11、
  vm_fault/pmap_enter/pmap_get_prot 命中 0。
  解读：fault handler 执行极快（微秒级），循环的慢在于
  "重试-再 fault"次数巨多，而非 handler 本身慢。
- rep movsb 目标页本次 0xffffffaaa8a19000（随 boot 变化）。
- 栈回溯/页表 walk 因 VM 提前 panic 未完成。

## 上游同步版 panic（复现）

- 同步上游（Rust c7c4183 + C shim 1c9c3aa3）后两次 boot 都在
  kext 列表加载完成后 panic（In Memory Panic Stackshot + corefile
  记录，serial 无 panic 文本）。
- 同步前（旧版）boot 可到桌面（bash-123 成功 + SSH + 干净关机）。
- 疑似同步引入回归（或旧版未曾触发的随机 panic——待回退验证）。
- QEMU 侧 panic 时刻 "failed to get xsave state"（噪声或线索）。
- corefile 在 guest 磁盘（下次 boot 可取，路径待查）。

## 下一步

- 回退 A/B 验证 panic 归因（旧版 vs 上游同步版）。
- 取 guest corefile 看 panic 栈。
- boot 慢：vm_fault 观察换 QEMU 侧 RIP 匹配软断点（强制 exit 后
  检查），或接受采样统计结论、转向 physmap 判定源码审查。

