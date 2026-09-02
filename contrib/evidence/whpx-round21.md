# 第二十一轮：meson stub 真因 + 上游同步 panic 解析 + A/B 回退验证（2026-09-03）

## 1. round20 "721 字节 stub" 的真正原因（不是 meson）

- deepseek 会话在 09-01 01:53 用 `git show pr2:hw/display/<f> > <windows-tree>/<f> 2>/dev/null`
  回退 C shim。`pr2` 不是可解析的 ref（只存在 `refs/tmp/pr2`），git 报错被吞，
  但 shell 的 `>` 已先把 4 个文件（pci.c / shim.c / shim.h / mmio.c）截断为 0 字节。
  空文件编译出 721 字节 obj，设备自然没注册 → "'reims-vgpu-pci' is not a valid device model name"。
- 同时，交接文档里"回退 C = 3aba725a (PR #2)"本身也不成立：3aba725a 与 1c9c3aa3 的
  hw/display 源完全相同，都已带 v20 ABI（`ReimsVgpuMapPagesFailure`），无法与 Rust 2835b6c0 配对。
- 正确的旧版配对由 gitlink 决定：`git ls-tree 2835b6c0 vendor/qemu` = **6118cb2b**
  （"reims-vgpu shims: enable Windows host builds"，父 e17ddb98，旧 `map_pages(..., out_ptr)` ABI）。
- 用户于 09-03 00:34–01:28 用 Codex 完成了还原：Windows 树现在 = Rust 2835b6c0（0 差异）+
  C shim 6118cb2b（0 差异），01:28 重建的 exe 经 `-device help` 确认已注册 `reims-vgpu-pci`。
  坏状态备份在 /tmp/reims-vgpu-windows-qemu-before-20260903。
- whpx-all.c（含全部诊断）自 09-01 00:09 起未变，因此本轮 A/B 的唯一变量就是 Rust + C shim。

## 2. 上游同步版 panic 的解析（serial 里其实有完整 panic 文本）

serial-upstream-sync-panic.log 第 962 行起（交接文档说"serial 无 panic 文本"不准确）：

```
AMFI: Denying core dump for pid 310 (SecurityAgent)
panic(cpu 3 caller 0xffffff800b5b1343): Kernel trap at 0xffffff800b4842fb, type 14=page fault
CR2: 0x0 ... RDI: 0x0  RSI: 0x0  RBX: 0xffffff8b6e57b980
Panicked task: pid 310: SecurityAgent (9 threads)
 _lck_spinlock_timeout_set_orig_ctid + 0x2fb   <- 实为 lck_spin_lock_grp 入口 `mov (%rdi),%rax`, rdi=0
 _knote_vanish + 0x57                           <- kqlock(knote_get_kq(kn)) 而 kq == NULL
 _ipc_mqueue_changed / _ipc_port_clear_receiver / _ipc_port_destroy
 _ipc_right_terminate / _ipc_space_terminate
 _task_mark_corpse / _proc_prepareexit / _exit / _postsig_locked / _bsd_ast
Kernel slide 0x0b0dc000, uptime 1933 s
```

- 解读：SecurityAgent（登录窗口认证 UI，第一个大量使用 GPU 的用户进程）先异常退出
  （收到致命信号 → 内核准备 core dump 被 AMFI 拒绝 → 转 corpse）；在拆除它的 IPC 空间时，
  某个 port 的 klist 里有一个 kqueue 指针已为 NULL 的 knote，`knote_vanish` 对它上锁 → NULL 解引用。
  这是 xnu 在"多线程进程异常退出 + machport knote 并发拆除"下的竞争，**触发条件是 SecurityAgent 崩溃**。
- 因此真正要归因的问题是"同步版下 SecurityAgent 为什么崩"，而非 panic 本身。
- 上游自己在 08-30 也记录了 x86 macOS / Linux Vulkan 通路上的**间歇性 guest panic**
  （ded2082："guest panics at 42 s ... controls died at 42/42/47 s"；98d66f5："killed its guest
  three times out of three, within ~2 s of a right-click on a dock icon"）。上游 c7c4183 带着这个
  未解的间歇缺陷。

## 3. A/B 回退验证（旧版组合，参数与 panic 两次 boot 完全一致）

- 启动命令：`QMP_PORT=4499 GDB=1 MACHDD_DEV="nvme,serial=macos" bash vm/boot-windows.sh`
- 02:00:05 启动；OpenCore 这次自动进入 Macintosh HD（Timeout=2），02:00:36 到 `kdp_core`；
  本次 kernel slide 0x0cc00000（PF-DIAG rip=ffffff800cd01057）。
- QMP `screendump` 在 GOP 阶段可用（shim 注册了 QemuConsole），/tmp/qmp.py + /tmp/shot.sh。
- 结果：（待填）

## 4. 工具

- /tmp/qmp.py：QMP 客户端（cmd / keys / shot）；/tmp/shot.sh TAG：截屏并缩放为 PNG。
- /tmp/guest-collect.sh：SSH 进 guest 拉 DiagnosticReports（*.panic、SecurityAgent*）与
  09-01 两次 panic 时段的 unified log。
