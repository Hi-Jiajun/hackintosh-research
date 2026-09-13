# range 开放问题裁决（docs/14 §8 四条）（2026-09-13）

> 本文对 `docs/14-资源范围与共享分配设计.md` §8（179–186 行）列出的四个开放问题逐个给出
> 可执行裁决。**只做裁决，不改代码**：本文只新增本文件，未触碰 `docs/11`（长期 dirty）与
> 任何其它 docs。结论不假设管理者偏好：能维持现状的给出"维持现状 + 触发条件"，需要改动的
> 明写代价与判别力损失。
>
> 路径约定：`crates/**`、`conformance/**`、`examples/**` 相对
> `/home/hiliang/hackintosh/metal-api-emulator`（分支 `shared-provider-objects`，本文撰写时
> HEAD `5d53cee`）；`docs/NN-*.md` 相对 `/home/hiliang/hackintosh/research`。
> 所有行号均可用 `rg -n` 复现，复现命令见 §5。

## 0. 复核后的现状（对任务背景的两处纠正）

1. **"第一版对重叠（含读读）一律拒绝"成立，但拒绝点只有两处门禁，不在 hazard 内核里。**
   读读不冲突的语义**已经实现并测试过**：`RangeSet::conflicts_with`
   （`crates/metal-api-core/src/provider.rs:3221`）、`RangeHold::conflicts`
   （`crates/metal-api-core/src/provider_api.rs:353`）与 trace 校验
   （`crates/metal-api-core/src/provider.rs:1252`）都只在"至少一侧可写"时报冲突。
   一律拒绝来自 admission 的 `DistinctViews` 分支（`provider.rs:2947`）与对象 API 的
   `set_buffer` 重叠检查（`provider_api.rs:1086`）。
2. **分配级 copy 计数契约不是"v10 专属"，而是"除 Swift oracle 外的每个 provider backend，
   且只在单次提交的用例上校验"。** 见 `conformance/compare.py:335`（`provider_backend`）、
   `compare.py:384`（v11 强制上报计数）、`compare.py:437`（单次提交用例的计数契约）；
   v10 只是当前**唯一**带"同一 allocation 多 view"的套件，因此判别力最强。契约边界的设计
   依据是 `docs/15` §5a–f（107–139 行）与 §5d（127 行，"Swift oracle 不参与"）。

## 1. 只读重叠是否放行

### 结论

**维持现状：不放行。** 同一 allocation 的第二个 view 与已有 view 只要字节重叠就继续
typed-refuse（`buffer_alias_unsupported`），读读也不例外；放行的前置条件与最小步骤见下，
在满足前不进入实现。

### 依据

- 设计意图：`docs/14:181` 把本条列为开放问题；`docs/14:55-68`（§3.1）把
  `DistinctViews` 的语义钉死为"两两 range 不重叠 ⇒ 允许；任何重叠（含读读）⇒ 第一版仍拒绝"。
- admission：`crates/metal-api-core/src/provider.rs:2947-2961` 在 `AliasMode::DistinctViews`
  下用 `views.values().any(|other| other.overlaps(&range))` 判定，**完全不看 access**；
  `BufferRange::overlaps`（`provider.rs:3184`）也只比较字节区间。
- 对象 API：`crates/metal-api-core/src/provider_api.rs:1086` 在 `set_buffer` 内对同一
  encoder 的任意第二个重叠 range 直接返回 `AliasedBufferBindings`，同样不看 access。
- 回归断言：`provider.rs:6816` 的 `distinct_view_ranges_are_admitted_only_while_disjoint`
  显式断言"读读重叠也被拒"（`provider.rs:6834-6846`）；对象 API 侧
  `crates/metal-api-core/src/provider_api/tests.rs:1232` 断言重叠多 view 提交被拒
  （该用例两个 view 都取自已写 pipeline，所以即使放行读读也仍然拒绝）。
- 内核已经就绪：`RangeSet::conflicts_with`（`provider.rs:3221`）、`ranges_conflict`
  （`provider_api.rs:359`）、trace 校验 `OverlappingWritableViews`（`provider.rs:1252-1268`
  ——条件里带 `other_access.is_writable() || view.access.is_writable()`）三处都已是
  "含写才冲突"。**因此放行是一次门禁放宽，不是 hazard 语义重写。**
- 若今天放行，会立刻撞上三处驱动/实现的硬约束：共享 backing 对同一 allocation 同一 offset
  的第二次上传直接报错（`crates/metal-api-vulkan/src/lib.rs:2331-2336`），字节计数会重复计
  重叠字节（`lib.rs:2348`），conformance 侧还有两道拒绝
  （`conformance/compare.py:146-153` 拒绝同 allocation 重叠初始化；
  `examples/metal-smoke/src/bin/provider-capture.rs:621-640` 拒绝同 allocation 重叠 range），
  并且 `compare.py:158-163` 把"同 allocation 多 view"限定给 v10 套件。

### 落地方式

现在：零改动。再评估的触发条件（三者同时满足才动手）：(a) 真实 wire trace 里出现"同一
backing 的重叠只读 view"（当前 v1–v11 fixture 里没有）；(b) 需要用它换取共享读的并行度，
且 `provider-smoke` 能给出可证伪的并行证据；(c) 下列最小步骤一次性做完。若决定放行，最小
步骤（每步独立可验证）：

1. admission 的 view 表从 `BTreeMap<ViewId, BufferRange>` 扩为携带 writable 位，判定改为
   "重叠且至少一侧可写 ⇒ 拒绝"，直接复用 `RangeSet::conflicts_with` 的规则；
   单测：读读重叠放行、读-写与写-写仍拒绝（改 `provider.rs:6816` 的断言）。
2. `set_buffer`（`provider_api.rs:1086`）的重叠判定加上同样的 access 条件；单测覆盖
   "同 encoder 两个只读重叠 view 允许、含写仍拒"。
3. 共享 backing 的上传改为按 range **并集**去重（把 `uploaded_ranges: BTreeMap<offset,len>`
   换成区间集合并），并把 `copy_in_bytes` 定义为并集字节数（`lib.rs:2331-2348`）。
4. conformance：新增 v12 套件（或把 v10 白名单扩到 v12）承载"同 allocation 两个重叠只读
   view"，`compare.py` 允许只读重叠初始化（初始字节必须逐字节相等，否则仍按写序依赖拒绝），
   `provider-capture.rs` 允许只读重叠 range，并同步更新 v10 白名单规则（`compare.py:158-163`）。
5. 探针：两个只读重叠 view 的两个 command buffer 必须同时在途。正反用例照抄
   `crates/metal-api-core/src/provider_api/tests.rs:985`（disjoint 必须并发）与
   `tests.rs:1022`（重叠必须串行），把前者改成"只读重叠也必须并发"。

### 影响面

- trace/wire：不变。沿用 `docs/14:63-66` 的实现决定（不新增 capability 字段，`MCC1` 帧不动）。
- 旧用例：不会变红。v1–v11 没有任何 fixture 使用同 allocation 的重叠 view，唯一需要改的是
  `provider.rs:6816` 那条"读读也拒绝"的断言。（这一条由 suite 加载器保证：
  `compare.py:146-153` 拒绝同一 allocation 的重叠初始化，因此现存 fixture 里不存在重叠
  multi-view 的用例。）
- 新代价：read 集合从"只用于 hazard 的辅助信息"变成可观察契约的一部分（`docs/14:181`
  已经预告了这个代价），且 `compare.py` / `provider-capture.rs` 两处驱动必须同步放宽，
  否则新套件无法构造。

## 2. `DistinctViews` 下 writeback 报告粒度

### 结论

**维持逐 view 报告**（`BufferWriteback { view_id, allocation_id, offset, bytes }`），
不按 allocation 合并。

### 依据

- 报告单元本身是 view：`crates/metal-api-core/src/provider.rs:2442-2447`；
  `docs/14:102-112`（§3.4）明确"报告格式不变，变的是 reservation 与 merge 的粒度"。
- compare 的逐 view 契约：`conformance/compare.py:291-304` 要求
  `expected_writebacks` 逐条覆盖**每一个可写 view**，且
  `(allocation, offset, len(data))` 必须等于该 view（`compare.py:300`）；
  `compare.py:363-367` 还要求实际 writeback 的**身份与顺序**与 suite 完全一致；
  `_writeback` 的字段集是精确匹配（`compare.py:77`），加/减字段会让所有旧 capture 失效
  （同类约束在 `docs/15:120-126` §5c 有记录）。
- allocation 级的真值**已经单独存在**，不需要合并 writeback 来表达：
  `compare.py:301` 把每条 writeback 叠回 `allocations[allocation]` 镜像，
  `compare.py:372-382` 要求 capture 逐个 allocation 上报 `bytes_hex` 并逐字节校验；
  `compare.py:437-443` 用分配级 copy 计数校验"每个被触及/被写的 allocation 恰好一次
  上下传"。也就是说分配级落地证据由"镜像 + 计数"承担，writeback 承担的是 view 级归属。
- v10 规则：`compare.py:158-163` 规定"同 allocation 多 view 只由 v10 套件资质"，
  而 v10 两个用例（`conformance/suite-v10.json`）的判别力恰好来自"每 view 一条 writeback
  + `copy_in=copy_out=1`"的组合；把 writeback 合并成 allocation 级会让 `compare.py:304`
  的"每个可写 view 都被覆盖"检查退化为同义反复。

### 落地方式

零改动。**若**将来仍要改成按 allocation 合并，代价必须先接受：

1. 删除/放宽 `compare.py:300`（writeback 必须精确覆盖某 view）与 `compare.py:304`
   （所有可写 view 都必须被覆盖）两条检查——这是判别力的净损失；
2. 字节判据只剩 `compare.py:372-382` 的 allocation 镜像与 `compare.py:437-443` 的计数，
   而镜像与计数在方法论上弱于"逐 view 给出字节"（`research/docs/15:66-73` 只把计数当作
   "共享成立"的证据，不当作字节证据）；
3. Swift oracle（`conformance/NativeOracle.swift`，见 `docs/15:127-131`）与四条 provider
   轨的 capture 全部要改格式，旧 capture 全部作废，五路径 compare 需要重跑重归档。

因此建议维持现状，并把"writeback 逐 view、allocation 级真值由 `allocations` 与计数承担"
这一条写进 `conformance/README.md` 的契约说明（属于文档动作，不在本文范围）。

### 影响面

- trace/wire：不变（写回也不走 `MCC1`，是 capture 报告字段）。
- 旧用例：不变。报告格式、顺序、canonical order（`docs/14:106-112`）都不动。
- 若走"合并"路线：v10 用例的判别力下降，且旧 capture 全废，属明确的负收益。

## 3. 与 `VK_EXT_external_memory_host` / 共享 section 的交互：range 是否要按页对齐

### 结论

**range 层不做页对齐，保持字节粒度。** 页对齐只属于"宿主映射/无拷贝导入"这一层，且只在
真正 import 时强制；对齐量必须取运行时的 `minImportedHostPointerAlignment`，不要写死
4096。共享 section 的基址已按系统页对齐，无需改动。

### 依据

- 类型层无对齐约束：`BufferRange`（`crates/metal-api-core/src/provider.rs:3160-3197`）是
  半开字节区间，`docs/14:69-89`（§3.2）的冲突规则也只比较字节。路线 A 已落地为
  "每 allocation 一个普通 device buffer"，创建走 `create_buffer` + HOST_VISIBLE 内存
  （`crates/metal-api-vulkan/src/lib.rs:2644-2720`），**不是** imported 内存，因此不受
  host-pointer 对齐约束。
- 导入路径才查对齐：扩展存在性在 `lib.rs:432-445` 探测，`min_imported_host_pointer_alignment`
  在 `lib.rs:475-481` 读入，`lib.rs:684` 暴露为 `external_memory_host_alignment()`；
  provider 在 `crates/metal-api-vulkan/src/compute_provider.rs:571-586` 用
  `host_pointer.is_multiple_of(alignment)` 校验，失败 slug 为 `lease_alignment_unsupported`
  （`compute_provider.rs:1251-1260`）；能力位也按同一条件挂上
  （`compute_provider.rs:150`、`compute_provider.rs:98`）。
- owner 侧已经承担页对齐：`HostRegion` 校验 page size 为 2 的幂、base 与 length 对齐
  （`provider.rs:248-268`），`borrowed_window` 强制 offset/length 按 page_size 对齐
  （`provider.rs:281-300`）；`research/docs/19:27`（§2.2）已把它定为决策：
  "窗口一律页对齐，GPU 侧字节精度由 view 的 offset/length 表达"。
- 共享 section：`crates/metal-api-ipc/src/shared.rs:39-41` 与 `shared.rs:227` 明确
  映射基址按系统页对齐（Unix `shm_open`/Windows `CreateFileMappingW`，`shared.rs:6-7`）。
- **本机实测（原始输出见 §5）**：本机两块 Vulkan 设备中，
  - `Dozen`（`Microsoft Direct3D12 (NVIDIA GeForce RTX 5060)`）**不暴露**
    `VK_EXT_external_memory_host`，也没有 `minImportedHostPointerAlignment`；
  - `llvmpipe`（Lavapipe，CI 用的 CPU 驱动）暴露该扩展 revision 1，
    `minImportedHostPointerAlignment = 0x00001000`（4096）。

  同时 `/usr/share/vulkan/icd.d/nvidia_icd.json` 存在但在本机枚举不出设备（无
  `/dev/nvidia*`），即本机 Linux 侧的 RTX 5060 只能经 Dozen 看到，而 Dozen 无该能力。
  真机的原生 NVIDIA Vulkan 驱动则可用：`evidence/windows-rtx5060-e891102-2026-09-08/manifest.md:43`
  记录 `alignment=4096`。

### 落地方式

1. 现在：零改动。保持 `docs/19:27` 的"窗口页对齐 + view 字节精度"两层分工。
2. 若路线 A 将来升级为"整段 allocation = 一段 imported 宿主映射"（guest RAM 直通），
   在 allocation 创建处把窗口向上取整到 `external_memory_host_alignment()`，并把未对齐的
   尾部字节定义为"不可映射、回退 `StagedLease`"，而不是放宽 range 语义；
   `BorrowedNoCopy` 能力位已经按"设备是否暴露扩展"门控（`compute_provider.rs:150`），
   无能力时 `docs/19:35-36` 的退化路径（`StagedLease`）已存在。
3. 需要补的测试：非对齐指针的拒绝（现成覆盖：
   `examples/metal-smoke/src/provider_suite.rs:1980` 的 `alignment={alignment}` 用例）；
   若引入按页对齐的 allocation 窗口，再补一条"对齐窗口 + 跨窗口 view 边界"的 smoke，
   并断言 `minImportedHostPointerAlignment` 是从设备读到而非写死的常量。

### 影响面

- trace/wire：不变（对齐是 provider 侧能力，不进 `MCC1`）。
- 旧用例：不变。v1–v11 的 allocation offset/length 都是 4 字节粒度且不导入宿主内存；
  `BorrowedNoCopy` 用例本来就要求 4096 对齐的宿主缓冲。
- 新用例：guest-memory 相关用例需要各自声明 page size（`HostRegion`/`DirtySet` 已带该字段，
  `provider.rs:224-231`、`provider.rs:340-345`）。

## 4. 与 queue priority / 公平性的交互：range hazard 是否是调度的第二维度

### 结论

**不是（不放行）。** 维持"hazard 归 admission（正确性）、队列选择归负载（性能）"的两分。
真正需要处理的是**等待者的唤醒公平性**（reservation 现在是
`notify_all` + 竞争重取锁，非 FIFO）；`docs/21` 的优先级契约应落在 reservation
的等待队列上，而不是队列选择器上。

> 依赖说明：`research/docs/21-队列优先级与公平性设计.md` 已于同日生成（2026-09-13），其 §5
> （91–102 行）自己就回答了本问题："只回答第一维度……将来把 hazard 升级成第二维度时，只需把
> hazard 冲突集合作为前置过滤，窗口公式不变"，并把"区间级 hazard 是否让同一档位的队列
> 进一步排序"列为未决。本文对该未决项的裁决是：**不做**，除非能给出可证伪的探针（见下）。
> 注意 `docs/21:127-139` 记录其实现在 `feat-priority-fairness` worktree 的 `4e15373` 上、
> 尚未合并到本文所引用的 `5d53cee`，所以 §4 的代码引用都指向主 checkout。

### 依据

- `docs/21:91-102`（§5）给出与本条一致的正交性论证与反向约束："`docs/14` 的
  whole-allocation 预约是**正确性**约束，优先级不得越过它……算法只在候选集合内部选择，
  不会把被预约挡住的队列重新拉回调度"。
- 两层已经分开且顺序固定：core 的 reservation 在 commit 前按 range 阻塞
  （`crates/metal-api-core/src/provider_api.rs:434-463` 登记、`provider_api.rs:524-547`
  聚合 pass 的 range），Vulkan 的队列选择发生在**真正提交时**
  （`crates/metal-api-vulkan/src/compute_provider.rs:965` 调 `pick_queue`），
  只看每队列在途计数（`crates/metal-api-vulkan/src/lib.rs:294-310` 的 `select_queue`、
  `lib.rs:525-532` 的 `pick_queue`、`lib.rs:320/324` 的计数器与锁）。
  **被 hazard 挡住的命令根本不会走到队列选择**，所以 range hazard 事实上已经是
  "准入维度"，再把它做成第二维度是重复表达。
- 队列优先级在设备创建时固定，运行期不可改（`lib.rs:405-419` 把每条队列的
  `queue_priorities` 设为 `1.0`；`docs/21:23-42` §2 解释了为什么不走这条路），
  provider API 也没有优先级字段
  （`HANDOFF-2026-09-09-QUEUE-SCHEDULING.md:85-86` 记录"先不伪装"）。
- 真正的公平性缺口在 reservation：释放时 `self.inner.available.notify_all()`
  （`provider_api.rs:487`，Drop 实现 478-489），等待方在 `reserve_ranges`
  （`provider_api.rs:437-446`）里被广播唤醒后竞争重取锁，**唤醒顺序无任何保证**；
  CPU 访问路径 `lock_unreserved`（`provider_api.rs:400-425`）同理。
  即"高优先级的冲突命令先拿到 range"这件事无法由队列选择器实现。
- 跨层约束：对象 API 通过 `&dyn PipelineProvider` 提交，core 看不到 queue index
  （同类约束见 `docs/15:133-136` §5e）；把 range 信息送进 `select_queue` 需要新增跨层接口，
  收益小于成本。`docs/21:60-90`（§4）的算法输入是"每队列在途计数 + 优先级 + 单调游标"，
  也没有 range 维度。
- 现行调度证据：`select_queue` 为"最少在途 + 轮询游标打破平局"，单测在
  `lib.rs:4707-4713`；多队列计数在提交/退休时增减（`lib.rs:547-563`）。

### 落地方式

1. 现在：零改动。`docs/21:91-102` 已经把"hazard 是前置过滤、不是调度维度"写进文档，
   本文确认该口径，不需要追加修改。
2. `docs/21:101-102` 把"区间级 hazard 是否让同一档位的队列进一步排序"列为未决；本文裁决为
   **不做**：只有当出现"同一档位内因 range 冲突导致队头阻塞、且优先级窗口无法缓解"的可复现
   证据时才重新评估，届时改动点仍在 core 的候选集合过滤，而不是队列选择器。
3. 若 `docs/21` 的优先级接线推进：先把 `Buffer::reserve_ranges` 的等待改成顺序确定的结构
   （按优先级选择唤醒对象，或 FIFO ticket + 定向 `notify_one`），再加可证伪探针；
   队列选择器保持"最少在途"不变，除非 `docs/21` 给出跨队列优先级的设备侧证据。
4. 需要补的测试：一个 gate-backed 探针，同时让两条命令等待同一个 range，断言唤醒顺序
   符合契约；当前实现应当**失败**（可证伪），否则说明探针没有真的制造竞争。

### 影响面

- trace/wire：不变（`docs/21:1-6` 也明确第一版不接 trace/wire）。
- 旧用例：不变。现有并发用例（`tests.rs:985`、`tests.rs:1022`）只断言"并发/串行"，
  不断言唤醒顺序，因此不会变红；`docs/21:127-139` 的实现也没有改 `select_queue`
  的现网行为。
- 新工作：优先级类型与唤醒策略属 `docs/21` 的交付物；本文只钉死"不在队列选择器里做
  range 维度"。

## 5. 抽样验证命令与原始输出

以下命令在本机（Linux 侧，`VK_ICD_FILENAMES` 未设置的默认配置）与仓库当前 HEAD 上原样执行
过，输出为原始粘贴。

### 5.1 admission 无视 access（问题 1）

```console
$ cd /home/hiliang/hackintosh/metal-api-emulator
$ rg -n "Ranged aliasing: several views|Overlap keeps the original refusal|Overlapping ranges keep the alias refusal|conservatively in the first version" crates/metal-api-core/src/provider.rs
2948:                            // Ranged aliasing: several views of one allocation
2950:                            // disjoint. Overlap keeps the original refusal so a
6834:        // Overlapping ranges keep the alias refusal even for read-read pairs,
6835:        // which stay refused conservatively in the first version.
```

```console
$ rg -n "pub fn conflicts_with" -A 8 crates/metal-api-core/src/provider.rs
3221:    pub fn conflicts_with(&self, other: &Self) -> bool {
3222-        self.writes.iter().any(|write| {
3223-            other.writes.iter().any(|other| write.overlaps(other))
3224-                || other.reads.iter().any(|other| write.overlaps(other))
3225-        }) || self
3226-            .reads
3227-            .iter()
3228-            .any(|read| other.writes.iter().any(|write| read.overlaps(write)))
3229-    }
```

（即：hazard 内核读读不冲突，拒绝来自 `provider.rs:2947` 的 admission 门禁。）

### 5.2 逐 view 的 writeback 契约（问题 2）

```console
$ rg -n "writeback does not cover exact writable view|expected writebacks do not cover writable views|several views of one allocation are only qualified" conformance/compare.py
161:                     f"{where}: several views of one allocation are only qualified "
300:                     f"{where}: writeback does not cover exact writable view {view}")
304:        _require(written_views == writable_views, f"{where}: expected writebacks do not cover writable views")
```

```console
$ rg -n "provider_backend = |copy_in \{counts\[0\]\} does not match \{expected_in\}" conformance/compare.py
335:    provider_backend = report["backend"] != "native-metal"
441:                     f"{where}: copy_in {counts[0]} does not match {expected_in} "
```

### 5.3 本机 `VK_EXT_external_memory_host` 可用性（问题 3）

```console
$ ls /usr/share/vulkan/icd.d/
dzn_icd.json  lvp_icd.json  nvidia_icd.json

$ VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/dzn_icd.json vulkaninfo 2>/dev/null | rg -n "deviceName|VK_EXT_external_memory_host|minImportedHostPointerAlignment|driverName"
177:	deviceName        = Microsoft Direct3D12 (NVIDIA GeForce RTX 5060)
404:	driverName                                           = Dozen

$ VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/lvp_icd.json vulkaninfo 2>/dev/null | rg -n "deviceName|VK_EXT_external_memory_host|minImportedHostPointerAlignment|driverName"
342:	deviceName        = llvmpipe (LLVM 22.1.8, 256 bits)
688:	minImportedHostPointerAlignment = 0x00001000
868:	driverName                                           = llvmpipe
1078:	VK_EXT_external_memory_host                        : extension revision 1

$ VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/nvidia_icd.json vulkaninfo --summary 2>&1 | tail -2
ERROR: [Loader Message] Code 0 : setup_loader_term_phys_devs:  Failed to detect any valid GPUs in the current config
ERROR at /usr/src/debug/vulkan-tools/Vulkan-Tools/vulkaninfo/./vulkaninfo.h:249:vkEnumeratePhysicalDevices failed with ERROR_INITIALIZATION_FAILED

$ ls /dev/nvidia* 2>&1 | head -1
ls: cannot access '/dev/nvidia*': No such file or directory
```

结论：本机 Dozen（RTX 5060 经 D3D12）无该扩展；Lavapipe 有，且对齐量为 4096。
真机证据（原生 NVIDIA 驱动）见 `evidence/windows-rtx5060-e891102-2026-09-08/manifest.md:43`
的 `alignment=4096`。

### 5.4 队列选择与等待唤醒（问题 4）

```console
$ rg -n "fn select_queue|pub\(crate\) fn pick_queue|queue_priorities|available.notify_all" crates/metal-api-vulkan/src/lib.rs crates/metal-api-core/src/provider_api.rs
crates/metal-api-core/src/provider_api.rs:487:        self.inner.available.notify_all();
crates/metal-api-vulkan/src/lib.rs:294:fn select_queue(in_flight: &[usize], round_robin_start: usize) -> usize {
crates/metal-api-vulkan/src/lib.rs:418:                    .queue_priorities(priorities);
crates/metal-api-vulkan/src/lib.rs:525:    pub(crate) fn pick_queue(&self) -> usize {
```

## 6. 影响面汇总

| 问题 | 本文裁决 | 是否动 trace/wire | 旧用例（v1–v11）是否变红 | 需要新增的测试（若推进） |
|---|---|---|---|---|
| 1 只读重叠 | 维持现状（不放行），附放行前置条件与 5 步 | 否 | 否（仅 `provider.rs:6816` 的读读断言需改） | 只读重叠并发 probe + 读-写仍拒绝 |
| 2 writeback 粒度 | 维持逐 view | 否 | 否 | 无（零改动）；建议补文档说明 |
| 3 页对齐 | range 保持字节粒度；对齐只在导入层，取运行时值 | 否 | 否 | 非对齐指针拒绝（已有）、对齐窗口 smoke（新增时） |
| 4 调度维度 | 不做第二维度（与 `docs/21` §5 一致）；公平性缺口在 reservation 的唤醒顺序 | 否 | 否 | gate-backed 唤醒顺序 probe（属 `docs/21`） |

## 7. 建议修正（不改本文档以外的文件）

1. `docs/14:90-101`（§3.3）仍把路线 A 描述为"推迟/另文"，而它已按 `docs/15` §5 步骤 2/3
   落地（`docs/15:191-231`）。建议在 §3.3 加一句指向 `docs/15` 的取代说明。
2. `docs/14:188-` 的 §9 末尾已声明"本节不再追加新状态"，但 §8 的四个开放问题没有回填裁决；
   建议在 §8 每条后加一行指向本文。
3. `docs/15:176` 的开放问题"分配级 copy 计数是否需要成为 `ProviderCapabilities` 的一部分"，
   实际已被 `e4911aa` 以"**不**新增能力字段、契约放在 `compare.py`"回答
   （`docs/15:141-153`、`docs/15:232-236`）。建议把该条从开放问题移到已完成项，
   并注明契约范围是"除 Swift oracle 外的 provider backend × 单次提交用例"。
4. `docs/14:181` 与本文 §1 的口径差异（"含读读拒绝"是门禁行为，不是 hazard 行为）建议在
   `docs/14` §3.2 补一句，避免后来者以为要改 `RangeSet`。

## 8. 待确认清单

1. `research/docs/21-队列优先级与公平性设计.md` 是**同批次并行产出**的文档（已在 research
   仓库提交为 `d4d5f13`），本文按该版本复核了 §4（引用 `docs/21:91-102`）；若 `docs/21`
   后续改动 §5 的口径，§4 的引用需要跟着更新。另外 `docs/21:127-139` 的实现提交 `4e15373`
   只在
   `worktrees/metal-priority-fairness`（`feat-priority-fairness` 分支）上、**未合并**到本文
   引用的 `5d53cee`，因此本文 §4 的代码行号都以主 checkout 为准。
2. 本机 Linux 侧的 RTX 5060 只能经 Dozen 看到，Dozen 不暴露 `VK_EXT_external_memory_host`；
   问题 3 中"RTX 5060 上可用该扩展"的结论**不是**在本机 Linux 上实测得到的，而是引用
   Windows 真机证据（`evidence/windows-rtx5060-e891102-2026-09-08/manifest.md:43`）。
   若要在本机复现原生驱动行为，需要 Windows 侧（或非 WSL 的 Linux 主机）重跑。
3. 问题 1 若放行，`copy_in_bytes` 的定义（并集字节数 vs 逐 view 字节数之和）需要与
   `docs/15` §5 的计数契约一起定稿，本文只给出倾向"并集"，未定稿。

## 9. 下一步建议（3 条以内）

1. 先做零风险的文档收口：按 §7 回填 `docs/14` §8/§3.3 与 `docs/15` §8 的过时口径，
   让"开放问题"只剩真正未决的项。
2. 在 `Buffer::reserve_ranges` 的等待路径上加"唤醒顺序可证伪"的探针（当前实现应当失败），
   把公平性问题暴露在 core 层；不要同时改队列选择器（`docs/21:91-102` 已排除第二维度）。
3. 若确有 wire trace 需要同 backing 的重叠只读 view，再按 §1 的 5 步一次性放行
   （含 v12 套件与 `compare.py`/`provider-capture.rs` 的同步放宽）；否则保持拒绝。
