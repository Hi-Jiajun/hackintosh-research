# 第二十二轮：最新上游同步 + boot 慢分析续（2026-09-03）

## 状态衔接

- round21（Codex）：stub 真因 = deepseek 会话回退命令 `>` 截断文件；
  panic 解析 = SecurityAgent 崩溃 → knote NULL 解引用（上游已知间歇缺陷）。
- 本会话接手：Windows 树同步到**最新上游**（Rust fd437422/747580f，
  483 提交；C shim bd88218 + _WIN32 munmap 守卫补丁），构建成功。

## boot 验证（最新上游版）

- boot 慢速阶段依旧（kdp_core 后 35+ 分钟）；
- 最终 panic：kernel_task 在 kext 区（0xffffff7fb106022e）NULL 写
  （err=2 写、CR2=0）——与 SecurityAgent 变体不同，属 WHPX 通路的
  NULL 解引用老模式（早期 round 的 ifnet 指针损坏同类）。
- 结论：panic 非同步回归（旧版/新版/最新版各有 panic 变体），
  WHPX 通路自身的历史债务；boot 慢独立于 panic。

## boot 慢分析（vm_fault 源码）

- 排除 recover_table：21 条全部在 0xffffff8000330dxx-0xfxx（copyin/
  copyout 的 bcopy 保护），不含 memcpy 核心 0xffffff8000101057。
- vm_fault_internal 主流程已读（lookup → prot → fault_page），
  "成功但不修页"的精确分支仍在定位中。
- 资产已迁 /home/hiliang/hackintosh/assets/（xnu-full + disasm，233MB）。

