# arXiv:2606.22283 第7章《Weights and compression》深度解读

> 论文：*Apple Neural Engine: Architecture, Programming, and Performance*
> 作者：Spencer H. Bryngelson
> 单位：Georgia Institute of Technology
> 提交：2026-06-21，302 页，12 图
> 许可：CC-BY 4.0
> DOI: `10.48550/arXiv.2606.22283`

---

## 0. 章节定位

第7章位于论文 **Part II "Reaching the ANE"** 的中段，前面是第6章《Dispatching without Core ML》（直连路径五步法），后面是第8章《Entitlement boundary》（授权边界）。

它回答一个核心问题：

> **训练好的几亿个权重，怎么从 DRAM 搬到引擎里乘加阵列门口？压缩能省的不是磁盘，而是这条搬运路上的字节流量。**

注意它和第25章《Compression internals》的分工：
- **第7章**：用户视角 —— 四种格式在每代芯片上**能不能流式**？速度涨多少？怎么选？
- **第25章**：内部视角 —— codec 字节布局、OCG 打包记录、稀疏双机制、FP8/Winograd 边界。

第7章是"地图"，第25章是"剖面图"。

---

## 1. 核心论点 (SUMMARY)

作者用一段话讲清了整章逻辑：

> **引擎在乘法器入口处把四种压缩权重现场还原成 fp16，再走和普通 fp16 权重完全相同的乘加通路。压缩省的不是磁盘，是 DRAM 带宽。**

然后是全章最关键的一句：

> **同一种格式在不同代芯片上命运不同：M1 只让 int4 LUT 和结构化稀疏"流式"过，int8 和块级仿射"折叠"回 fp16；M5 上四种全部流式。**

这把读者最大的误区纠正了：压缩格式**不是你声明了就省带宽**，而是**芯片代际决定它走哪条路**。同一段 int8 权重，在 M1 上一字节没省，在 A14 上直接砍半。

---

## 2. 一句话讲清"压缩"到底压缩了什么

神经网络的算力其实不是瓶颈，**带宽才是**。一个卷积层 / 矩阵乘，每次推理都要把整份权重从 DRAM 搬到引擎，权重一动就是几 MB 到几十 MB。算得再快，搬运慢也白搭。

所以论文反复强调：

> **权重流（weight stream）是任何"算术强度低"的层的主要开销，这覆盖了几乎所有 decode 形状和 projection 重的工作。**

把权重压成 int4 / int8 / 稀疏，**不是为了省磁盘**，而是**为了让字节流变小、DRAM 搬运变快**。这是整章的物理动机。

---

## 3. 两种结局：流式（stream）vs 折叠（fold）

全章最关键的二分法。一个压缩格式在芯片上有两种命运：

| 结局 | 描述 | 带宽收益 |
|---|---|---|
| **Stream（流式）** | 压缩字节直接进引擎，**在芯片里**现场解压回 fp16，再喂给乘法器 | **有**：搬运字节减少 → 层更快 |
| **Fold（折叠）** | 进入 dispatch 之前**就在 DRAM 里**被还原成 dense fp16 常量 | **无**：搬运的还是 fp16 字节，没有任何带宽节省 |

**关键判断**：是 stream 还是 fold **不由你声明决定，由目标芯片决定**。同一份 int8 权重：
- M1：fold（DRAM 里展开成 fp16 再搬，没省）
- A14 起：stream（搬的就是 int8 字节，在乘法器门口解）

这个二分是全章最重要的实操结论。

---

## 4. 四种压缩格式 (§7.2)

### 4.1 全景

引擎认识且仅认识四种压缩格式，**和 Core ML Tools 里文档化的"模型体积压缩"用的是同一套**：

| 格式 | 数学形式 | 一行 constexpr MIL op |
|---|---|---|
| int8 仿射（per-tensor / per-channel） | `w = s·(q − z)` | `constexpr_affine_dequantize(q, scale, zero_point)` |
| int4 查找表（palette / LUT） | `w = lut[indices]` | `constexpr_lut_to_dense(indices, lut)` |
| 结构化稀疏 | mask + 非零值 scatter | `constexpr_sparse_to_dense(mask, nonzeros)` |
| 块级仿射（blockwise） | 每个 block 一个 scale | `constexpr_blockwise_shift_scale(q, scale)` |

**关键工程细节**：每个格式在 IR 里是**一个**折叠到 weight descriptor 里的 constexpr op，**不是**一个独立的 backend operation。每种格式的 op 数量固定，**多一个或少一个都是硬校验错误**。

### 4.2 int8 仿射量化（线性量化）

最朴素的形式。每个权重存 1 字节，配上一个 fp16 scale（可选每个输出通道一个）：

```
编码：q = round(x / s) + z
解码：w_fp16 = scale * (q - zero_point)
```

**M1 的特殊简化**：在 H13（M1）一代，零点 `z` 折叠为 0，所以 M1 上的 int8 只支持**对称**形式 `w = s·q`。后续代才支持完整的仿射（含零点）。

> 通俗类比：原来你存一个 2 字节的浮点数，现在只存 1 字节的"刻度位置"+ 一个公共比例尺。精度从 ±0.0002 退到 ±0.01，但体积砍半。

### 4.3 int4 查找表（palette / LUT）

和 int8 完全不同——它**不是算术量化**，而是**调色板**。

- 每个权重存 4 比特（半字节），是一个 **0–15 的索引**
- 另外有一张 16 项的 **fp16 codebook**（查找表）
- 解码 = `w = codebook[index]`，**纯查表，无算术**

**论文里给的一个具体例子**（值得记住）：

权重 `[1.0, 0.0, 0.0, 1.0]` 配一张 16 项 codebook，前两项是 `0.0 (entry 0, 0x0000)` 和 `1.0 (entry 1, 0x3c00)`。索引流 `[1, 0, 0, 1]` 按"低 nibble 在前"两两打包成一个字节：

```
索引 [1, 0]  →  字节 0x01   (低 4 位是 1，高 4 位是 0)
索引 [0, 1]  →  字节 0x10
```

所以 4 个权重只占 2 字节，**相对 fp16（8 字节）压缩 4 倍**。解码时每个 nibble 当作表索引查回 fp16 值。

> 通俗类比：GIF 图像的调色板。图像里每个像素不存 RGB，只存"颜色编号 0–255"，再配一张 256 色的调色板。int4 LUT 就是"16 色调色板"版的权重。

### 4.4 结构化稀疏（structured sparsity）

针对"权重里有一半以上是 0"的情形：

- 存 1 比特 mask 标记"哪些位置是非零"
- 后面跟着非零值的 fp16 紧凑打包

重建 = 把非零值 scatter 回 mask 标记的位置，**精度无损**（除 fp16 自身舍入）。

成本估算：如果一个权重 50% 是 0，mask 占 `N/8` 字节，非零值占 `N/2 × 2 = N` 字节，总共 `1.125N` 字节，**约 dense fp16 的 56%**。零越多越省。

> 通俗类比：地铁高峰期统计乘客座位。"10100101" 这样的 mask 告诉你哪些座位坐了人，后面只列有人的座位号和姓名，不用把整节车厢的座位都报一遍。

### 4.5 块级仿射（blockwise affine）

int8 的精细化版本。不是每个 tensor / 每个输出通道一个 scale，而是**每个连续小块**（比如每 16 或 32 个元素）一个独立 scale。

代价：scale 数量增多，元数据变大。
收益：量化误差更低，因为每个小块的数值范围更窄、scale 更贴合。

数学形式 `w = s_block · (q − z)`，每个 block 自己一组 `(s, z)`。

---

## 5. 谁流谁折：每代芯片的命运表 (§7.3)

### 5.1 Table 7.1（最值得记住的一张表）

| 格式 | M1 / H13 | A14 / M2 | A15 / M3 及以后 | M5 / H17s | 实测加速 |
|---|---|---|---|---|---|
| int8（per-tensor / per-channel） | **fold** | stream | stream | stream | M5 1.6–1.8×；M1 无收益 |
| int4 LUT | **stream** | stream | stream | stream | M1 2.37×；M5 1.6–1.8× |
| 结构化稀疏 | **stream** | stream | stream | stream | M1 1.55–1.64× @ 0.43× bytes；M5 1.6–1.8× |
| 块级仿射 | **fold** | **fold** | stream | stream | M5 1.6–1.8×；M1/M2 无收益 |

**一句话读法**：M1 上只有 int4 和稀疏**真能省带宽**，int8 / 块级只省磁盘。从 A14 起 int8 也开始流式。从 A15 起全部四种都流。M5 上四种格式齐齐 1.6–1.8× 加速。

### 5.2 谁决定了 stream vs fold？是 HAL feature byte

**这个二分不是任何单个 op 的属性，而是 hardware-abstraction-layer (HAL) 的决策**。每个权重相关 op 在每颗芯片上都合法，区别在于"流式 master 门 + 一组 per-format 门"是否打开。

Table 7.2 给出了具体 feature byte：

| HAL byte | 作用 | M1/H13 | A14/M2 | A15/M3 | A16/A17/M5 |
|---|---|:-:|:-:|:-:|:-:|
| `+0x48f` | kernel-streaming **master** | 1 | 1 | 1 | 1 |
| `+0x528, +0x532, +0x537` | per-format gates（**A14 开启**） | 0 | 1 | 1 | 1 |
| `+0x520, +0x523, +0x533, +0x539` | per-format gates（**A15 开启**） | 0 | 0 | 1 | 1 |
| `+0x529` | palette & stride gate | 1 | 1 | 1 | 1 |

**怎么读**：
- master 门从 A13（含 M1）就是 1 → "允许权重流式"
- palette 门在 M1 上也是 1 → 所以 int4 LUT 能流（int4 走 palette 路径）
- A14 才打开的那组门 → 让 int8 / 稀疏开始流
- A15 才打开的那组门 → 让块级仿射开始流
- 稀疏走的是**另一条路**：它是 mask-and-values 的 weight operand，归 master 门管，不依赖 per-format 那组。**所以 M1 上稀疏也能流**。

> 这是论文反复出现的 motif：**同一个特性在不同代芯片上由不同的 feature byte 把守**，软件必须按目标芯片单独验证。

### 5.3 一个特别需要记住的反直觉事实

**M1 上的 int8 是"省磁盘不省带宽"**。

- 磁盘上：0.5× fp16 大小（确实小一半）
- DRAM 搬运时：**展开成 dense fp16**，仍然是 fp16 的字节数
- 结果：matmul 延迟 = fp16 延迟，**没有任何加速**

这就是 Table 7.4 里 `int8 fold on M1: 1.0× fp16 latency` 的含义。

从 A14 起，int8 才**真正以 int8 形态被 dispatch**，在乘法器入口处解。Table 7.3 给出了 A14/M2 上的实测：

| 权重宽度 K=N | fp16 延迟 | int8 延迟 | int8 / fp16 |
|---:|---:|---:|---:|
| 2048 | 0.290 ms | 0.253 ms | 0.87 |
| 4096 | 0.740 ms | 0.447 ms | 0.60 |
| 8192 | 2.610 ms | 1.351 ms | **0.52** |

权重越宽，int8 越接近理论极限的 0.5×。因为权重越大，搬运越占主导，压缩收益越明显。

---

## 6. 实操决策树 (§7.4, §7.6)

### 6.1 论文给的硬规则

按目标芯片决定：

- **M1 / H13**：
  - 权重 ≥ 50% 是零 → **结构化稀疏**（流式，无损，1.55–1.64×）
  - 否则 → **int4 LUT**（最密，能流，2.37×）
  - 16 级精度太粗、需要更细 → **int8**（仅省磁盘，不省带宽）
- **A14 / M2 起**：int8 也流了，可以放心用
- **M5 / H17s 及以后**：四种全流，**按精度/字节比挑**，不再按"能不能流"挑

### 6.2 为什么低精度流式安全？回到第3章

这是论文刻意安排的呼应：

> **第3章讲过，ANE 的乘加器有"宽累加器"**。一个流式进来的 int8 / int4 权重，**在乘法器入口处被还原成 fp16**，然后走的是**和 dense fp16 完全相同的乘加通路**。所以唯一的精度损失是权重量化本身，**累加精度一点没丢**。

这对卷积、矩阵乘、归一化这种"partial sums 留在范围内"的层是安全的。对**消项严重（cancellation-heavy）**的层（比如残差里大量正负抵消），**任何**低精度形式（包括 fp16 自身）都不安全，这跟第3章的数值讨论一致。

### 6.3 自动选择的伪代码

论文 §7.6 给了一个 `choose_weight_form` 函数，逻辑很直白：

```python
tolerance = 0.01   # 相对 fp32 参考的最大允许误差

def native_streams(chip):
    if chip == H13:  return [int4, sparse]      # M1: int8 和 blockwise fold
    if chip >= H14:  return [int4, sparse, int8]

def choose_weight_form(layer, weights, chip):
    if not is_bandwidth_bound(layer, chip):
        return fp16                                # 算力受限，压了也没用
    candidates = native_streams(chip)
    if fraction_zero(weights) < 0.5:
        candidates.remove(sparse)                  # 不足一半零，稀疏不划算
    for form in sorted(candidates, key=bytes_per_weight):  # 字节最少的先试
        if accuracy_error(form, weights, layer) <= tolerance:
            return form
    return fp16                                    # 全都不达标就保持 fp16
```

**核心思想**：只在带宽受限的层挑格式；按"字节最少优先"尝试；每种都过一遍精度关，过了就用；都不过就退回 fp16。

---

## 7. 一个特别实用的技巧：热补丁 (§7.5)

**已编译的程序可以在不重编译的前提下热替换权重值**。

原理：
- 每个 weight tensor 在 program image 里占一块**已知 tiling 的 decoded region**
- 卷积权重用 `0xC0-stride` 布局
- 矩阵乘权重用 `0x40-stride` 布局
- 改权重值**不动 descriptor**，也**不破坏 op 绑定**
- 所以同一份编译好的 program 可以直接换上新系数继续跑

驱动侧有专门的 per-patch mutable-buffer 记账路径：`ANEScheduler::pendingRequestsPerPatchMutableBuffer`。

> 通俗类比：印好的报表模板，数字格子里填什么数都行，不用重新排版印刷。这对**LoRA 微调权重热替换、A/B 测试不同 checkpoint、在线学习权重更新**都是巨大利好——省掉一次 daemon compile + sign。

---

## 8. 参考速查表 (§7.7, Table 7.4 / 7.5)

### 8.1 每代实测（精选）

| 常量 | 代 | 值 |
|---|---|---|
| int4 LUT 流式加速 | M1/H13 | **2.37× fp16** |
| 结构化稀疏流式加速 | M1/H13 | 1.55–1.64× fp16 |
| 结构化稀疏存储字节 | M1/H13 | 0.43× dense |
| int8 在 M1 上（fold） | M1/H13 | **1.0× fp16**（无收益） |
| int8 磁盘体积 | M1/H13 | 0.5× fp16 |
| int8 精度（vs fp32） | M1/H13 | cosine ≈ 1.0，相对误差 ≈ 1% |
| int8 matmul 2k 宽 | A14/M2 | 0.85× fp16 |
| int8 matmul 8k 宽 | A14/M2 | **0.52× fp16** |
| int8 流式加速 | A14/M2 | 0.64× fp16 |
| 结构化稀疏流式加速 | A14/M2 | 0.54× fp16 |
| 块级仿射 fold | A14/M2 | 0.985×（几无收益） |
| 四种格式全部流式 | M5/H17s | **1.6–1.8× fp16** |
| 流式 master 门 | A13 起 | on |
| int8 per-format stream 起点 | A14 | M2 silicon 上确认 |
| 块级仿射 per-format 起点 | A15 | 预测值，未 silicon 确认 |

### 8.2 元素类型目录（Table 7.5）

引擎认识的元素类型里**和权重相关**的子集：

| 类型 | 宽度 | 在权重里的角色 |
|---|---|---|
| int8 | 1 字节 | int8 仿射 lane |
| uint8 | 1 字节 | uint8 仿射 lane |
| float16 | 2 字节 | dense 权重 / codebook 项 / scale / bias |
| uint4 | 4 比特 | 4 比特 palette 索引 |
| int4 | 4 比特 | 4 比特有符号，**M1 上仅作 palette 用** |
| e4m3 | 1 字节 | fp8，**M1 上被门控关闭** |

**关键细节**：datapath 里**没有 int4 算术 lane**，所以 4 比特值永远是 palette 索引，指向 16 项 fp16 codebook。这就是为什么 M1 上 int4 标记为 "palette-only"。

---

## 9. 三个最值得记住的判断

1. **"压缩 ≠ 省带宽"**。一个格式是否省带宽，**由目标芯片的 HAL feature byte 决定，不由你声明决定**。M1 上的 int8 就是反例：磁盘小一半，DRAM 搬运一字节没省。

2. **"fold 是格式存在，stream 才是格式有效"**。fold 形式在工程上几乎等同于"你写了一个更小的 .mlmodel 文件，但运行时还是跑 fp16"。只在 stream 形式下，压缩才真正转化为延迟降低。

3. **"压缩安全性来自宽累加器，不是来自低精度算术"**。ANE 永远把压缩权重**先还原成 fp16**，再走 fp16 乘加 + 宽 reduction。所以累加精度不丢，丢的只是权重本身的量化误差。这也是为什么低精度流式在卷积、matmul、归一化上是安全的。

---

## 10. 与其他章节的勾连

- **→ 第3章（Numerics）**：宽累加器是压缩权重安全性的物理基础；fp16 datapath 决定了"还原点必须在乘法器入口"
- **→ 第5章（Software stack）/ 第6章（Dispatching without Core ML）**：压缩权重走的是 direct route，**不**受 entitlement 门控，reconstruction op 不需要特权
- **→ 第8章（Entitlement boundary）**：第7章是"压缩格式不走 entitlement"的实例，与第8章的"四项被门控特性"互为对照
- **→ 第24章（HAL and capability gates）**：本章 Table 7.2 的 feature byte 是第24章 HAL 表的具体实例
- **→ 第25章（Compression internals）**：本章是用户视角，第25章是同主题的内部视角（codec 字节布局、OCG 打包、双稀疏机制、FP8/Winograd 边界）
- **→ 第9章（Roofline）**：本章 §7.6 的 `is_bandwidth_bound` 判断直接对应 roofline 模型里的"带宽受限 vs 算力受限"分界

---

## 11. 速查图（一张图记住全章）

```
┌────────────────────────────────────────────────────────────────────┐
│              压缩权重的"一生"                                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  训练好的 fp32/fp16 权重                                              │
│         │                                                          │
│         ▼  Core ML Tools 量化/调色板化/稀疏化                          │
│  ┌──四种压缩格式──────────────────────────────┐                       │
│  │ int8 仿射   w = s·(q−z)                    │                       │
│  │ int4 LUT    w = codebook[idx]              │                       │
│  │ 结构化稀疏  mask + nonzeros scatter        │                       │
│  │ 块级仿射    每 block 一组 (s, z)            │                       │
│  └────────────────────────────────────────────┘                       │
│         │                                                          │
│         ▼  每种格式 = 一个 constexpr MIL op，fold 进 weight descriptor │
│                                                                    │
│  ┌── HAL feature byte 决定走哪条路 ──────────────────┐                │
│  │                                                    │                │
│  │  master +0x48f ─┬─► per-format gates (A14 开)      │                │
│  │  (A13 起为 1)   │                                  │                │
│  │                  └─► per-format gates (A15 开)      │                │
│  │                  └─► palette gate +0x529 (M1 就开)  │                │
│  │                                                    │                │
│  │     ↓ 门开                  ↓ 门关                  │                │
│  │   STREAM                  FOLD                      │                │
│  │   压缩字节直送             先在 DRAM 展开成 fp16      │                │
│  │   乘法器门口解压            搬运仍按 fp16 算           │                │
│  │   带宽省 ✓                 带宽没省 ✗                 │                │
│  └────────────────────────────────────────────────────┘                │
│         │                                                          │
│         ▼                                                          │
│  在乘法器入口处统一还原成 fp16                                       │
│  → 进入和 dense fp16 相同的乘加 + 宽累加器                              │
│  → 累加精度无损，只损失权重量化误差                                     │
└────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│              每代芯片谁流谁折                                          │
├──────────────┬────────┬────────┬────────┬────────┬─────────────────┤
│ 格式         │ M1/H13 │ A14/M2 │ A15/M3 │ M5/H17s│ 备注             │
├──────────────┼────────┼────────┼────────┼────────┼─────────────────┤
│ int8 仿射    │ fold   │ stream │ stream │ stream │ M1 上省磁盘不省带宽│
│ int4 LUT     │ stream │ stream │ stream │ stream │ M1 上 2.37×      │
│ 结构化稀疏    │ stream │ stream │ stream │ stream │ 走 mask 路径     │
│ 块级仿射      │ fold   │ fold   │ stream │ stream │ A15 起才流       │
└──────────────┴────────┴────────┴────────┴────────┴─────────────────┘
```

---

## 12. 给工程师的实操要点

如果你正在为 ANE 优化模型，这一章转化为几条可以直接落地的建议：

1. **先判目标芯片，再选格式**。同一份 .mlmodel 在 M1 和 M5 上行为完全不同，部署前一定要在目标芯片上测延迟，而不是只看磁盘大小。

2. **M1 上：优先 int4 LUT 或结构化稀疏**。它们是 M1 上唯二真正省带宽的格式。int8 在 M1 上只适合"磁盘紧张但带宽不紧张"的场景。

3. **A14+ 上：放心用 int8**。从 A14 起 int8 真正流式，权重越宽收益越大（8k 宽时延迟砍到 0.52×）。

4. **稀疏只在零占比 ≥ 50% 时考虑**。低于这个比例，mask 自身的开销盖过了非零值省下的字节。

5. **能用 LoRA / 权重热补丁就别重编译**。第7.5 节的 weight patching 路径让你直接改 program image 里的权重值，省掉一次 daemon compile+sign。

6. **算力受限的层别压**。`is_bandwidth_bound` 先判，算力受限的层压了也白压，反而可能伤精度。

7. **消项严重的层（残差累加、大动态范围 softmax 差）保持 fp16**。宽累加器救不了权重自身已经被量化掉的信息。

---

*本文档基于 arXiv:2606.22283v1 (21 Jun 2026) 第7章 (pp. 41–46) 撰写。论文为逆向工程参考文档，direct route 未被 Apple 官方支持、跨版本脆弱，仅适用于研究、测量和设备端实验，不应用于发布软件。Core ML 仍是唯一受支持的发行路径。*
