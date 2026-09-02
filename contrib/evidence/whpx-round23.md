# 第二十三轮：GPF 注入拿到完整 backtrace——boot 慢根因实锤（2026-09-03）

## 方法

- 主线程 stall watchdog（1s tick 采样 CPU0 stale RIP；内核区 RIP 10s 不变
  → 写 pending #GP + WHvCancelRunVirtualProcessor 强制 VP 停 → QMP cont
  deliver）。
- 关键坑：运行中 VP 写 pending event 不 deliver（无 exit 无 VM entry 时机），
  必须 cancel + 下次运行才 deliver。

## Backtrace（slide 0x15000000）

```
PE_init_iokit + 0x254
  -> (0xffffff8015cdcb33 / 0xffffff8015cdcc9f, kext loader 内部)
  -> OSKext::addKextsFromKextCollection + 0x1e / + 0x24e
  -> rep movsb @ memcpy 核心 0xffffff8000101057
```

- 业务：IOKit 从 kernelcache 装载 kext 时 memcpy 16KB kext 数据
  （rsi=0xffffff8017ab2000 源）到**新分配的 kalloc 页**
  （rdi=0xffffffc4ab9d8000 目标），该页未映射 → #PF 无限重试。
- 与之前所有观测一致：vm_fault "成功"但 PDPTE 恒 0、rcx 恒 0x4000。

## 下一步（决定性实验）

- 循环中 QMP stop 后，gdb 手动在 guest 页表里建立目标页的映射
  （任选一个空闲物理页），cont 看循环是否打破：
  打破 → 页表缺失是唯一卡点，根因收敛到 vm_fault 的 pmap_enter 路径；
  不打破 → 还有其他机制。
- 或深挖 vm_fault 对 kernel_map submap（zone）fault 的递归路径。

