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
