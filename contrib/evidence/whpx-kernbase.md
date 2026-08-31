# WHPX 深栈取证（2026-08-31 第二轮）

## 停止点

第一条 #PF 停止（PF-DIAG < 2 + vm_stop），QMP 4468。

- RSP=0xffffff8017598f10 RBP=0xffffff8017598ff0 CR3=0x1c0ae000
- RSI=0x1c0ae000 == CR3（格式串指针=CR3 的损坏模式再次实锤）
- slide = 0x174dc000（movsbl 模式定位）

## RBP 链新增两帧

d24: ret=bcopy_phys+0x8e   （高半区内核代码）
d25: ret=0x400027a0        （identity 映射低地址！）

d25 是当前已知最深的帧：返回地址指向 identity 映射区
（物理 0x400027a0），而 dump 显示 0x4000000-0x4040000 全零。

## 内核物理基址

mach-o magic 搜索定位：物理 kernBase = 0x176dc000 =
slide(0x174dc000) + 0x200000（符合 OpenCore kernBase=slide+0x200000 约定）。

## 未初始化的内核全局（panic 时）

vm_kernel_slid_base/top/slide/stext/top 在 panic 时刻均为
OpenCore plist 的 ASCII 残留（如 rPincGhG、ertSwipe、struc</k）——
即 vm_init 之前这些全局从未被初始化，进一步确认 panic 发生在
XNU bootstrap 极早期。

## 待对照

KVM（WSL 通路，正常 boot）下物理 0x4000000 区域是否有内容：
- 全零 → 0x400027a0 是坏返回地址（XNU 早期某返回地址计算错误）；
- 有内容 → WHPX 下该 bootstrap 区域未正确填充。