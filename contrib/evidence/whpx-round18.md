# 第十八轮：CR3 flush 实验未加速 boot（2026-08-31）

## 实验

- 实现 QEMU 主线程 100ms timer 对全部 VP 写 CR3（同值）强制 guest
  TLB flush；初版用 WHvGet 直读运行中 VP 的 CR3（stale）写回导致
  QEMU 退出；改为主线程用 env->cr[3] 缓存（每次 VP 退出时同步）。
- 结果：QEMU 稳定（set=ok），boot 到 kdp_core 3 分钟（与基线相当），
  但 8 分钟后 serial 仍停 kdp_core——慢速阶段未加速。

## 结论

- TLB stale 假设未证实：同值 CR3 写可能被 Hyper-V 优化（无 flush），
  或循环根因不在 TLB。
- 最硬证据仍是：vm_fault 返回成功但页表不变（PDPTE=0）——指向
  vm_fault 走了 physmap/reserved 判定路径（不修页表）。

## 下一步

- gdb 断点 vm_fault 入口/出口（guest 内核符号 + slide），观察
  "成功返回但页表不变"的具体分支（physmap 判定 vs reserved entry）；
- 或读 fault 地址所在 vm_map（physmap 区验证）。
