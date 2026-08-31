# 第十五轮：PR 整理与上游同步（2026-08-31）

## PR #3 整理（steelbrain/qemu-reims-vgpu）

- 旧分支从 reims shim 分支分叉，PR diff 混入 40+ 个 shim 提交；
  已 rebase：6 个 whpx 修复 cherry-pick 到上游最新 master
  （300438f），worktree 方式，主 checkout 与 kvm.c 脏改动未动。
- 上游 711 个新提交中的 API 变更（cpu_single_stepping() 等 4 个
  whpx/accel 相关）自动适配，无冲突。
- misc_enable 提交作者修正为 Jiajun Liang。
- PR 描述更新为 6 修复完整版 + base 选择说明（accel 层独立于
  shim，target master；PR #1/#2 先例 base=host-reims-vgpu-vmapple
  是设备修复）。
- PR 状态：MERGEABLE，6 commits，待 review。

## 上游动态

- steelbrain/qemu-reims-vgpu：master=纯 QEMU 上游；
  host-reims-vgpu-vmapple=shim 分支（基 b8337166，落后上游 711）。
- steelbrain/reims-vgpu：活跃；**PR #57（windows-host-port）已
  merge**（2026-08-31 06:14 UTC）——Windows 移植已进上游。
- 用户 fork：qemu fork master 已同步；Rust fork master 有 101
  提交拓扑差异但无独有内容（建议网页 Sync fork，开 PR 用特性分支）。

## GitLab 4385

- 补进展 note（3763368137）：6 修复 + PR 链接 + 剩余问题。

