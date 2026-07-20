# 第24章补充：算子校验流程深度剖析

> 论文：*Apple Neural Engine: Architecture, Programming, and Performance*
> 作者：Spencer H. Bryngelson（Georgia Institute of Technology）
> 提交：2026-06-21，302 页，CC-BY 4.0
> 本章原文：pp. 157–163（§24.2、§24.8）
>
> 这是 [ch24 深度版](./arxiv-2606.22283-ch24-hal-capability-gates.md) / [精炼版](./arxiv-2606.22283-ch24-hal-capability-gates-condensed.md) 的专题补充，聚焦"一个算子从被声明到真正在硅上跑，要经过哪些校验层"。

---

## 0. 为什么单独拎出来

论文 §24.2 给了两道闸门（MinimumFamily + capability byte），§24.8 又强调"attested is not reachable"。但论文没有把**完整校验流水线**画出来。把它拼齐后会看到：**一个算子从 IR 到硅上跑，要过 6 道关，挡在哪一道决定了能不能绕**。

---

## 1. 完整校验流水线

```
源代码 / IR
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│  ①  Frontend 识别                                          │
│  "编译器前端认不认识这个算子？"                              │
│  · 不认识 → 编译错误（unknown operation）                  │
│  · 认识 → 继续                                             │
│  例：3D conv 前端识别（要求 static weight）→ 通过           │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────────────────────┐
│  ②  HAL 标量上限检查                                       │
│  hal[scalar_offset] 对照                                   │
│  · 张量宽度、3D 核深度、SRAM 工作集等                       │
│  · 超 HAL 表里的值 → 拒绝或拆分                            │
│  例：3D conv 核深度 0x70=16 通过；张量 >16384 → 拒绝        │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────────────────────┐
│  ③  闸门 A：MinimumFamily<N>（算子级 trait）                │
│  target_family >= op.minimum_family ?                      │
│  · 否 → 分解为合法的更基础算子（decompose）                 │
│  · 是 → 继续                                               │
│  例：sin floor=A15，M1=family2 → 分解；M5=family6 → 原生    │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────────────────────┐
│  ④  闸门 B：HAL 能力字节（hal[offset] & 1）                 │
│  控制算子内部路由                                          │
│  · 0 → 走软件分解路径                                      │
│  · 1 → 走原生硬件路径                                      │
│  例：纹理引擎 0x81d，M1=0 → resize 走软件分解              │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────────────────────┐
│  ⑤  Validator 通过？                                       │
│  · 某些算子有专门 validator（top-k、sort、dynamic-slice）  │
│  · validator callable 不等于能跑                           │
│  · validator reject → 编译错误                             │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────────────────────┐
│  ⑥  Backend lowering + 代码生成                            │
│  · 没有任何 backend 为它生成代码 → 拒绝                     │
│    "Not implemented on any of the specified backends"      │
│  例：3D conv 在所有 device mask 上 lowering 失败 ⚡         │
│  例：top-k/sort/dynamic-slice validator 过了但 codegen 拒   │
└──────────────┬─────────────────────────────────────────────┘
               │
               ▼
        ✓ 真正能在硅上跑
```

---

## 2. 六道关口的详细说明

### ① Frontend 识别

- **位置**：编译器 IR parser / 前端 type-checker
- **干什么**：把源代码或 IR 里的算子节点解析成编译器内部表示
- **失败表现**：`unknown operation` / `parse error`
- **典型例子**：
  - 3D conv：前端**识别**（且强制 static weight，错误消息 `"3D Convolution does not support dynamic weights"`）
  - `pixel_buffer_to_tensor`：IR parser **识别**，`pixel_buffer` surface type、`FMT_*` 格式 token、grammar、format enum、type rule **全部可达**
  - symbolic `?` 维度：**能 parse**，symbolic 变量 `s0` 能 bind

> **关键**：前端识别**只是承认语法合法**，不保证后续能编译。

### ② HAL 标量上限检查

- **位置**：编译器 legalization / shape-check 阶段
- **干什么**：把算子声明的张量形状、kernel 维度等数字，对照 HAL 表里的标量上限
- **失败表现**：shape 超限错误，或自动拆分（tiling）

HAL 表里的关键标量（精选）：

| 偏移 | 字段 | M1 值 |
|---|---|---|
| `0x70` | `max_large_conv_kernel_dim_z`（3D 核深度） | 16 |
| `0x138` | `max_tensor_width` | 16384 |
| `0x158` | `max_tensor_depth` | 16384 |
| `0x1b8` | `max_operand_bytes`（SRAM 工作集） | 2 MB |

> **陷阱**：3D conv 在这一层**通过**——HAL 0x70 写着 16，声明核深度 ≤ 16 完全合法。这是"attested but not reachable"的第一道伪装。

### ③ 闸门 A：`MinimumFamily<N>`（算子级 trait）

- **位置**：MLIR op trait `mlir::OpTrait::anec::MinimumFamily`
- **干什么**：每个 backend op 自带一个 `minimum_family` 属性，目标芯片的 family index 必须 ≥ 它
- **失败表现**：算子被重写成更基础的合法 op（decompose）

**家族 index 排序**：

```
A11Legacy=0  A12=1  A13=2  A14=3  A15=4  A16=5  A17=6  ...
                     ↑             ↑
                     M1            M5
```

**伪代码**（论文 Listing 24.2）：

```python
def op_is_native(op, target_family):
    return target_family >= op.minimum_family
    # 例：softmax N=2 (A13)，sin N=4 (A15)

def lower_op(op, target_family):
    if op_is_native(op, target_family):
        emit_native(op)     # 一个 anec op
    else:
        decompose(op)       # 重写成 floor 以下的合法 op
```

**按 floor 分层**（论文 Table 24.2）：

| Floor | 原生起点 | 代表算子 |
|---|---|---|
| **F0** | 全部（A11Legacy 起） | 卷积、matmul、池化、elementwise、reshape、transpose、concat |
| **F2** | A13 起（含 M1） | softmax、LayerNorm / InstanceNorm / BatchNorm、reductions、attention、erf、sqrt |
| **F3** | A14 起 | crop-resize、resample（**纹理引擎相关**） |
| **F4** | A15 起 | sin、cos、global argmin / argmax |
| — | — | **没有计算算子 floor 高于 A15** |

> **重要观察**：A16/A17/A18 这些新代**不增加新算子**，只增加核数和频率。所以从 op floor 角度看，A15 已经是"算子集天花板"。

### ④ 闸门 B：HAL 能力字节 `hal[offset] & 1`

- **位置**：编译器里读 `ZinIrHalParameters` 的内联 `ldrb` 指令
- **干什么**：控制**同一个算子内部的一条路由**——比如有没有硬件纹理引擎
- **失败表现**：算子虽然 floor 够，但走软件分解路径

**关键能力字节**（论文 Table 24.3，精选）：

| 偏移 | 含义 | M1 | A14 | A15 | A16 | A18 |
|---|---|:-:|:-:|:-:|:-:|:-:|
| `0x48f` | kernel-streaming master（64 KB ↔ 16 MB 切换） | 1 | 1 | 1 | 1 | 1 |
| `0x529` | palette stream（int4 LUT 流式） | 1 | 1 | 1 | 1 | 1 |
| `0x81d` | **纹理引擎**（resize / crop-resize / affine / gather / 对称 padding） | **0** | 1 | 1 | 1 | 1 |
| `0x52d` | fp8 E4M3 | 0 | 0 | 0 | 0 | 1 |
| `0x563` | FIFO-mode DMA | 0 | 0 | 0 | 0 | 1 |
| `0x815` | 原生 softmax | 1 | 1 | 1 | 1 | 1 |

> **M1 的最大功能性缺口**：`0x81d = 0`。resize、crop-resize、resample、affine transform、硬件 gather、对称 padding **全部走软件分解**。

### ⑤ Validator 通过？

- **位置**：算子特定的 validator 函数（`Zin*ValidationUtils::Validate*`）
- **干什么**：对某些算子做专门检查（shape、range、layout 等）
- **失败表现**：validator reject → 编译错误

**陷阱**：validator **callable 且通过 ≠ 能跑**。M1 上 `top-k`、`sort`、`dynamic-slice` 的 validator 都 callable、都通过，但 codegen 阶段拒绝。

### ⑥ Backend lowering + 代码生成

- **位置**：MLIR lowering passes → 任务描述符生成
- **干什么**：把高层 op 翻译成引擎能执行的任务描述符
- **失败表现**：`"Not implemented: Some ops are not supported on any of the specified backends"`

**这一层失败的著名例子**：

| 算子 | 失败原因 | 后果 |
|---|---|---|
| 3D conv | 所有 device mask 都无 lowering | M1 上引擎跑不了，Core ML 直接放 CPU/GPU |
| `pixel_buffer_to_tensor` | direct route 不 lower | 直连图像格式输入不可达 |
| symbolic `?` 维度 | direct runtime path 不 lower | 必须编译一组 fixed shape |
| `top-k` / `sort` / `dynamic-slice` | validator 过，codegen 拒 | M1 上不可达 |

> **这一层是"硬墙"**——前端认识、HAL 写着支持、validator 通过，但 backend 不发代码。软件无法绕。

---

## 3. 两道闸门的真值表

闸门 A（MinimumFamily）和闸门 B（capability byte）**必须同时通过**才算"原生发射"：

| MinimumFamily（A） | capability byte（B） | 结果 |
|---|---|---|
| 家族够格 | 能力字节 = 1 | **算子原生发射**（一个 anec op） |
| 家族不够格 | — | 分解为更基础的合法算子 |
| 家族够格 | 能力字节 = 0 | 走软件分解路径 |
| — | — | 都不达标且有合法分解 → 分解 |
| — | — | 都不达标且无合法分解 → **编译拒绝**，错误消息点名架构 |

**实例**：
- M1 上的 `sin`：A 失败（floor=A15，M1=A13）→ **分解**为合法基础 op
- M1 上的 `resize`：A 通过（floor=A14，但...）→ 实际上 A 也失败，但即使 A 通过，B 也失败（纹理引擎=0）→ **软件分解**
- M5 上的 `sin`：A 通过（floor=A15，M5=A17），B 无关 → **原生发射**

---

## 4. 失败在哪一层决定能不能绕

**这是论文最重要的工程含义**——失败层越靠后，软件越没法绕：

| 失败层 | 能绕吗 | 绕法 / 例子 |
|---|---|---|
| ① 前端不识别 | ✓ 改写 IR | 用户拼错 op 名 → 修语法 |
| ② HAL 标量上限 | ✓ 改 shape | 张量宽度超 16384 → tiling / 拆分 |
| ③ MinimumFamily 不够 | ✗（per-chip 硬限制） | sin 在 M1 上 → 等下一代芯片 |
| ④ 能力字节 = 0 | △ 半自动（编译器已走分解） | resize 在 M1 上 → 软件分解已自动 |
| ⑤ Validator 拒 | ✗ 通常没法绕 | 边界 shape 非法 → 改 shape |
| ⑥ Backend lowering 失败 | **✗ 完全没法绕** | 3D conv / symbolic shape / `pixel_buffer_to_tensor` |
| ⑦ Codegen 拒（validator 通过后） | **✗ 完全没法绕** | top-k / sort / dynamic-slice 在 M1 上 |

**⑥⑦ 是真正的硬墙**——前端认识、HAL 写着支持、validator 通过，但 backend 不发代码。

> 论文给出的唯一可靠判据：**"compile-and-run on the target"**。第4章的"原生可用算子表"全部是 M1 上实测结果，不是从 HAL 表推断的。

---

## 5. 三条最值得记住的判断

1. **"Attested is not reachable"**（§24.8）—— HAL 表写着支持只证明**读取它的那一层**承认支持，**不证明**后续 lowering / codegen 会真的生成代码。3D conv 是教科书级反例。

2. **两道闸门独立**——MinimumFamily 管"算子级别"，capability byte 管"算子内部路由"，必须**同时**通过。M1 上的 resize 是闸门 B 单独卡住的典型（即使 MinimumFamily 通过）。

3. **失败层决定可绕性**——前端层、shape 层的失败可以靠改 IR / 改 shape 绕；validator 通过之后的 lowering / codegen 失败**完全没法绕**，是 per-chip 硬墙。这就是为什么论文反复强调"compile-and-run on the target"。

---

## 6. 与其他章节的勾连

- **→ 第4章（Capability surface）**：第4章的"原生可用算子表"是**实测**结果，本章 HAL 表 + MinimumFamily 是**声明**结果——两者不等价正是 §24.8 的主题
- **→ 第8章（Entitlement boundary）**：第8章的四项门控特性（3D conv / native stateful / bf16 / symbolic shape）都失败在**第⑥层**，这就是为什么 entitlement 撼动不了它们
- **→ 第7章（Weights and compression）**：第7章 Table 7.2 的 per-format 流式门是本章能力字节的具体实例
- **→ 第26章（Hidden layers & netplist）**：直连路径手写 `.espresso.net` 的合法性也是这条流水线验证，只是入口换成了直接 IR 提交

---

*本文档基于 arXiv:2606.22283v1 (21 Jun 2026) 第24章 (pp. 157–163，主要是 §24.2、§24.8) + 第8章 §8.2 的失败实例撰写。论文为逆向工程参考文档，所有失败位置和错误消息均在 M1/H13 上实测，仅供研究、测量和设备端实验使用。Core ML 仍是唯一受支持的发行路径。*
