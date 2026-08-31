# 第十三轮：WHPX macOS boot 成功（2026-08-31）

## 里程碑

- 修复 6（IA32_MISC_ENABLE 0x1a0）生效后，Windows WHPX 通路
  macOS boot 到达 launchd / APFS 事务 / AppleKeyStore /
  bluetoothd / MobileSoftwareUpdate 阶段——系统服务全面运行。
- **16 个 vCPU 全部进入内核 idle 循环**（0xffffff801edb3406,
  sti;hlt;cli 模式）——AP 启动完成。之前 round5-11 的
  "AP 从未启动"（0x7f96b034 OpenCore hlt）是 boot 卡在更早
  阶段的症状，不是独立 bug（G0/G2 假设均不成立）。
- 修复 6 已作为第 6 个 commit 提交到 whpx-fixes 分支并 push,
  PR #3（steelbrain/qemu-reims-vgpu）现含 6 个修复。

## rep movsb #PF 循环重新定性

- 无 gdb 单步污染的干净 boot 同样经历该阶段：CPU0 在 memcpy
  核心（unslid 0xffffff8000101057, rep movsb）写 0xfffffffd 区
  目标页时 #PF（err=2 写），guest handler vm_fault 处理,重试,
  最终通过并继续推进——**是慢速阶段而非死锁**。
- Hyper-V 对该阶段后续 #PF 直接 deliver 给 guest（不 exit 到
  QEMU）→ QEMU 零感知 → rippoll 无输出（"无退出"假象）。
- 15 秒级停滞源于 IF=0 长阶段 + guest 页表建立慢，具体性能
  机制待查（候选：Hyper-V 对 guest invlpg/页表缓存刷新的延迟）。
- 定性：性能问题，非正确性问题。不影响 boot 到达用户态。

## KVM 对照结论

- WSL 旧构建（11.0.5, whpx-fixes build 目录）boot Boot0002
  循环 = 旧 QEMU 版本问题；qemu-build.sh 重建（11.0.50）后
  OpenCore 正常。
- KVM + rails 镜像：CPU0 在 userspace/kernel 间正常切换、
  多 CR3——健康 boot，无 rep movsb #PF 循环 → 该循环 WHPX 特有。
- WSL reims-vgpu 窗口（dzn）D3D12 Removing Device → segfault：
  AndrowsVm（安卓模拟器）GPU-PV 冲突嫌疑，与 QEMU/KVM 无关；
  vmware-svga（gtk 窗口）可绕开。

## 遗留

- guest 后续进展观察：桌面/登录界面、reims-vgpu GPU 驱动加载、
  画面输出。
- rep movsb 慢速阶段的性能根因（可选优化）。
- PR #3 等待上游 review。

