# Upstream issue draft: `air.read_texture_2d` with a dynamic thread position

> 目标仓库：pinned metal2vulkan `43c46ac`（`metal-api-emulator` 的 workspace 依赖）。
> 本文是可直接提交的 issue 草稿；最小复现、期望行为与实测证据都在 `docs/16` §4.5 的
> 诊断链里。**提交前需要复验一次**（见 §4）。

## 1. 现象

把一个 kernel 的纹理读取坐标改成依赖 thread position 后，**只有前几个 invocation 读到
正确的 texel**，其余 invocation 读到 texel 0。相同地址计算、不含纹理读取的 kernel 在同一
执行路径与两种 launch 形态下都完全正确。

## 2. 最小复现

两个 AIR 模块只差一次纹理读取（其余完全相同）：

- **A（正确）**：`cell = 4*y + x`，`store cell`，无纹理参数。4x4 grid（local 4x4 与 1x1）
  都得到 `[0,1,...,15]`。
- **B（错误）**：同样的地址计算，但写入值是 `texel(x,y) + 100`。4x4 grid（两种 local）
  得到 `[100,101,102,103,100,100,...]`——前四个 invocation 读到 `texel 0..3`，其余 12 个
  读到 `texel 0`。

完整 AIR 见本仓库 `research/docs/16` §4.5 与探针记录；关键点：

```llvm
%coord0 = insertelement <2 x i32> undef, i32 %x, i64 0
%coord  = insertelement <2 x i32> %coord0, i32 %y, i64 1
%texel  = call { <4 x i32>, i8 } @air.read_texture_2d.u.v4i32(
            ptr addrspace(1) %texture, ptr addrspace(2) %sampler,
            <2 x i32> %coord, <2 x i32> zeroinitializer, i32 0, i32 0)
```

纹理是 4x4 `R32Uint`，值 0..15；host-visible 线性 `VkImage`，host 侧一次上传。

## 3. 期望行为

16 个 invocation 各自读到自己的 texel，输出 `[100,101,...,115]`（与纯 buffer 版本一致）。

## 4. 已排除项（复验清单）

| 嫌疑 | 结论 | 证据 |
|---|---|---|
| provider 特化/dispatch 参数 | 正确 | `METAL_API_DEBUG_DISPATCH=1`：`spec data=[4,4,1]`、`local=[4,4,1] groups=[1,1,1] base=[0,0,0]` |
| 生成的 SPIR-V 语义 | 正确 | `spirv-dis`：`gl_GlobalInvocationID` + region base、坐标 x/y 提取、`OpImageFetch ... Lod 0` |
| 驱动特有 | 不是 | Lavapipe 与 RTX 5060 都复现（子集不同） |
| 写回可见性 | 不是 | dispatch 后加 `COMPUTE→COMPUTE` 屏障无变化 |
| thread position 分量 | 正确 | 直接输出 x/y 的探针在两驱动上给出合理值 |
| 显式 group/local 坐标替代方案 | 不可用 | 被本项目的 footprint 证明拒绝（translator 无法表达该地址式） |

## 5. 求助

1. 这是 translator 在"纹理读取 + thread position"组合下的已知限制，还是缺陷？
2. 若有推荐的替代写法（不依赖 `GlobalInvocationID`、且能被 footprint 证明表达），
   请指路——`metal-api-emulator` 的 conformance 需要在 v11 引入多 invocation 的纹理用例。

## 6. 状态

- 本仓库当前保留**单 invocation** 的纹理用例（已通过，Lavapipe + RTX 5060）；
- 多 invocation 纹理用例在 issue 有结论前不进入 v11；
- 提交 issue 前请按 §4 复验清单在当前 pinned 版本上重跑一次，并附 `docs/16` §4.5 的原始输出。

## 7. 2026-09-14 复验结果：不提交上游 issue（归因被推翻）

按 §4 在当前 pinned 版本（metal2vulkan `43c46ac`，`git fetch` 后 `origin/master == 43c46ac`）
上重跑，得到：

1. **现象逐字复现**（Lavapipe / `llvmpipe (LLVM 22.1.8, 256 bits)`）：同一地址计算、去掉
   纹理读取的对照在 4x4 grid 的 local 4x4 与 local 1x1 两种形态下都是 `[0..15]`；
   只把写入值改成 `texel(x, y) + 100` 就得到 `[100,101,102,103,100,100,...]`，与本文 §2
   记录完全一致；`thread_position_in_grid` 的 x/y 分量成对写出也完全正确（§4 第 5 行成立）；
   显式 group/local 坐标式仍被 provider 的 footprint 证明拒绝（§4 第 6 行成立）；
   `METAL_API_DEBUG_DISPATCH=1` 显示特化与 dispatch 参数正确（§4 第 1 行成立：local 4x4 为
   `data=[4,4,1] groups=[1,1,1] base=[0,0,0]`，local 1x1 为 `data=[1,1,1] groups=[4,4,1]`）。

2. **但归因不成立**：新增的两个对照把缺陷定位到本仓库的 Vulkan provider，而不是
   translator：
   - 用 pinned `43c46ac` 的 CLI（`--stage kernel --local 1,1,1
     --threads-per-grid-push-constant 0`，与 provider 的 `TransformOptions` 同参数）翻译最小
     复现，`spirv-dis` 显示坐标是 `(x + base.x, y + base.y)`、写回下标是 `4y + x`，逐
     invocation 正确——§4 第 2 行成立；
   - **单 invocation + 常量坐标**读取 (0,0)(0,1)(0,2)(0,3)(2,1) 得到 `[0,0,0,0,0]`（期望
     `[0,4,8,12,6]`）。这里没有多 invocation，也没有坐标构造链，所以"translator 把坐标折叠
     为 0"不可能解释它。

3. **真实根因**：`crates/metal-api-vulkan/src/lib.rs` 的 `create_textures` 上传段假定线性
   镜像的行紧密排列（`row_pitch = width * 4`），从不查询驱动给的行距。Lavapipe 上 4x4
   `R32Uint` 线性镜像的 `VkSubresourceLayout.rowPitch` 是 **64 字节**（紧密排列只有 16），
   于是宿主把第 r 行写在偏移 `r*16`，驱动却按 `r*64` 读——只有第 0 行重合，其余行读到从未
   写入的偏移（0）。在仓库外的源码副本里只把落行改成按 `get_image_subresource_layout`
   的 `rowPitch`/`offset`，多 invocation 纹理读取在两种 launch 形态下立刻得到
   `[100..115]`，常量坐标探针也恢复为 `[0,4,8,12,6]`。

**结论**：这不是 metal2vulkan 的缺陷，不应向上游提交本文的 §1–§5。§4 排除表中的"生成的
SPIR-V 语义正确"仍然成立，但"缺陷在 translator 的纹理 intrinsic 降级路径"这一结论错误；
`docs/16` §4.5 的最终结论同样需要按本节修正。

**边界与后续**：

- 复验只用了 Lavapipe；RTX 5060 未重跑（需要交叉编译与 MSYS2 工具链），§4 第 3 行的
  "两驱动都复现"只能用既有归档输出推断（NVIDIA 的 `[0,1,0,1,8,9,8,9,0,...]` 与"行距 32
  字节"一致）；
- §4 第 4 行（写回可见性屏障实验）没有重跑：需要改 provider 代码，与"只读复现"冲突；
- 修复落在本仓库（尊重 `VkSubresourceLayout`），随后 v11 可以解除"多 invocation 纹理用例
  不进入"的限制；现有单 invocation 用例只读 texel (0,0)，永远不会暴露该问题；
- 完整证据、探针工程与原始日志：`evidence/upstream-issue-metal2vulkan-2026-09-14/`。
