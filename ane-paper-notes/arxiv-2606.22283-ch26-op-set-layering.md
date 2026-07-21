# 第26章补充：190 / 45 / Zin IR 底层 op 的三层算子目录关系

> 论文：*Apple Neural Engine: Architecture, Programming, and Performance*
> 作者：Spencer H. Bryngelson（Georgia Institute of Technology）
> 提交：2026-06-21，302 页，CC-BY 4.0
> 本章原文：pp. 165–169（§26.1–§26.3）
>
> 这是 [ch26 深度版](./arxiv-2606.22283-ch26-hidden-layers-netplist.md) / [精炼版](./arxiv-2606.22283-ch26-hidden-layers-netplist-condensed.md) 的专题补充，专门回答一个问题：**"Core ML 190 种 op 真的压缩成 45 种 IR op 吗？"** 答案是**否**——三个数字测的不是同一根坐标轴。

---

## 0. 为什么单独拆出来

精炼版里有一张图：

```
Core ML（≈190 op）→ 翻译器 → Zin IR（45 descriptor）→ 后端
```

这张图读快了会得到三个错误印象：
1. "190 全部压成 45"——**错**，190 是全后端目录，不是全部上 ANE
2. "45 是 Zin IR 的算子总数"——**错**，45 只是 netplist 可见的高阶 layer descriptor 数
3. "45 比 190 小，所以是裁剪"——**错**，45 里还藏着 190 里**根本没有**的高阶融合积木（SDPA / Sort / 点云 / 流式状态 / LUT）

把三层目录拆开看，才能正确理解 ch26 反复强调的"隐藏算子"到底是什么意思。

---

## 1. 三个数字测的不是同一根轴

| 数字 | 测的是什么 | 层级 | 谁能看到 |
|---|---|---|---|
| **≈190** | Core ML 对外暴露的**全后端 op 目录**（CPU + GPU + ANE 共享） | 框架层 | 任何 Core ML 调用方 |
| **45** | `_ANEC<Name>LayerDescInitialize` 认识的**高阶 layer descriptor 数** | Zin IR 高阶层 | 手写 netplist 的人 |
| **更多** | Zin IR 后端 lowering 出来的**细粒度 op**（DMA / barrier / tile / gain-offset / MAC chain ...） | Zin IR 低阶层 | 只有 ANECompiler 内部 |

**关键判断**：
- 190 和 45 不是"输入 vs 输出"——190 里的多数 op 根本不上 ANE
- 45 和"Zin IR 全部 op"不是一回事——45 之下还有一层
- 45 ⊄ 190，190 ⊄ 45——两者是**交集非空**的两个集合，且**双方各有独占元素**

---

## 2. 三个集合的韦恩图

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│   Core ML ≈190 op                  Zin IR 高阶 descriptor      │
│  ┌───────────────────┐            ┌──────────────┐             │
│  │                   │            │              │             │
│  │  CPU/GPU 专属     │            │  隐藏算子    │             │
│  │  (不上 ANE)       │            │  SDPA / Sort │             │
│  │                   │            │  点云 / 状态 │             │
│  │                   │            │  流 / LUT    │             │
│  │                   │  ← 交集 →  │              │             │
│  │                   │            │              │             │
│  └───────────────────┘            └──────┬───────┘             │
│                                          │                     │
│                                          ▼                     │
│                                Zin IR 低阶 op（lowering 展开）  │
│                                DMA / barrier / tile /          │
│                                gain-offset / MAC chain / ...   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 2.1 交集（Core ML 和 Zin IR descriptor 都有）

常规卷积、池化、elementwise、激活、concat 等。**这一块是 Core ML 路径 A 实际跑的东西**。

### 2.2 Core ML 独占（Zin IR descriptor 里没有）

- 表格类、循环控制类 op：Core ML 第一站就路由到 CPU
- 部分 reduction / scatter 类：路由到 GPU
- 自定义层（custom layer）：永远走 CPU

> 这些 op **不参与**"翻译到 ANE"——翻译器在 router 阶段就把它们踢出 ANE 候选。

### 2.3 Zin IR descriptor 独占（Core ML 目录里没有，"隐藏算子"）

论文 §26.3 + Table 26.1 列出的 7 个家族：

| 家族 | 代表 descriptor | Core ML 等价物 |
|---|---|---|
| 融合注意力 | `SDPA` | **无**（Core ML 必须拆成 4 个 op） |
| 排序与选择 | top-k / sort / argmin-max | 无独立 sort |
| 几何与点云 | cross product / FPS / radius search | 无 |
| 流式状态 | live state / ring-buffer / tensor-to-buffer | 无（外显成额外 I/O） |
| 可编程激活 LUT | `kZinIrNonLinearCustomLUT` | 无 |
| 空间重排 | pixel-shuffle / space-and-batch | 部分 |
| 纹理采样器 | resize / crop-resize / affine | 无原生 |

> 这就是 ch26 "45 < 190 但还藏着东西"的真正含义：**不是 IR 把 op 砍少了，而是这层有 Core ML 没暴露的高阶融合积木**。

### 2.4 Zin IR 低阶 op（不在 45 里）

`_ANEC<Name>LayerDescInitialize` 层级之下，ANECompiler 后端还会把每个 descriptor **lowering** 成一组细粒度 op：

- DMA move（含 4 个 kernel DMA 子通道：bias / post-scale / palette LUT / activation LUT）
- barrier / sync
- tile / untile / concatenate
- gain-offset 阶段
- MAC chain（卷积拆出来的乘加序列）
- LUT apply / scatter-gather

> **这些 op 不暴露给 netplist 作者**——`Unit.Type` 字段填不进去。它们是 ANECompiler 在 lowering 阶段自动生成的。

---

## 3. 翻译器真正做的事：多对多映射

不是"190 → 45"的一对一压缩，而是**多对多映射 + 部分回退**：

```
Core ML op（ANE 候选子集）
        │
        ▼  Core ML → Zin IR 翻译器
┌─────────────────────────────────────────────────────┐
│                                                     │
│  模式 ① 多对一（折叠）                                │
│  例：MatMul + Mul(scale) + Softmax + MatMul          │
│       ↓（理论上）                                    │
│       一个 SDPA descriptor                          │
│  ⚠️ Core ML 路径 A 不会自动做这种融合——                              │
│     只有直写 .espresso.net 才能触发 SDPA             │
│                                                     │
│  模式 ② 一对多（拆分）                                │
│  例：大卷积 → split-legalize → 一串小 conv descriptor │
│  例：某激活 op → conv + activation LUT + gain-offset   │
│                                                     │
│  模式 ③ 一对一                                       │
│  例：常规 conv / pooling 多数情况                     │
│                                                     │
│  模式 ④ 回退（不翻译）                                │
│  例：op 不在 ANE 可达集合 → 静默退回 CPU/GPU          │
│                                                     │
└─────────────────────────────────────────────────────┘
        │
        ▼
   Zin IR 高阶 descriptor 序列
        │
        ▼  ANECompiler 后端 lowering
   细粒度 op 序列（DMA / barrier / tile / ...）
```

### 3.1 一个具体例子：SDPA 的两种命运

同一个 Transformer block 里的 attention 计算，走两条路径得到完全不同的 Zin IR：

| 路径 | Core ML 表达 | Zin IR descriptor 序列 | 中间张量回写次数 |
|---|---|---|---|
| **A（Core ML 翻译）** | `MatMul → Mul(scale) → Softmax → MatMul` 四个独立 op | 4 个 descriptor（或更多，若被拆分） | 3 次 |
| **B（手写 netplist）** | 直接写 `Type = "SDPA"` 一个 Unit | 1 个 `SDPA` descriptor | 0 次 |

> **关键**：路径 A 的翻译器**不会自动融合到 SDPA**——Core ML 这边没有 "SDPA op" 可供融合。SDPA 是路径 B 独占的"隐藏 descriptor"。

---

## 4. 一张表浓缩三层关系

| 维度 | Core ML ≈190 op | Zin IR 45 descriptor | Zin IR 低阶 op |
|---|---|---|---|
| **层级** | 框架层（全后端） | 高阶 layer | lowering 后细粒度 |
| **谁能看见** | 任何调用方 | netplist 作者 | ANECompiler 内部 |
| **覆盖后端** | CPU + GPU + ANE | 仅 ANE | 仅 ANE |
| **包含隐藏算子** | 否 | 是（SDPA/Sort/点云/状态/LUT） | — |
| **是否暴露在 .espresso.net `Unit.Type`** | — | 是 | 否 |
| **是否被 Core ML → Zin IR 翻译器自动触发** | — | 交集部分是；隐藏算子**不是** | — |
| **数量级** | ≈190 | 45 | > 45（论文未给出精确数） |

---

## 5. 修正后的 ch26 路径图

把上面的层次关系叠回 ch26 路径图里：

```
            Core ML 模型（.mlpackage）
                    │
                    │  框架层 op 目录 ≈190 种（CPU/GPU/ANE 共享）
                    ▼
        ┌──────────────────────────────┐
        │  路径 A：Core ML → Zin IR     │  ← 翻译路由（sanctioned）
        │       翻译器                  │  · 多对多映射 + CPU/GPU 回退
        │  （折叠 / 拆分 / 回退）        │  · ★ 不自动融合到 SDPA
        └──────────────┬───────────────┘
                       │
                       │  翻译结果只命中
                       │  "Core ML ∩ Zin IR descriptor" 交集
                       ▼
                Zin IR 高阶 layer descriptor（45 种）
        ┌──────────────────────────────┐
        │  · 交集部分：conv/pooling/... │
        │  · 隐藏部分：SDPA/Sort/点云/  │  ← 路径 B 独占触发
        │    状态流/LUT 等              │
        └──────────────┬───────────────┘
                       │
                       │  路径 B：手写 .espresso.net
                       │  ↓ 可直接指定任何 45 个 Type
                       │  ↓ 包括隐藏 descriptor
                       ▼
            ANECompiler 后端 lowering
        ┌──────────────────────────────┐
        │  把高阶 descriptor 展开成      │
        │  细粒度 op 序列：              │
        │  DMA/barrier/tile/gain-offset │  ← 这一层算子数 > 45
        │  /MAC chain/...               │  ← netplist 作者看不见
        └──────────────┬───────────────┘
                       │
                       ▼
                  .mlmodelc → AppleH11ANEInterface → ANE 固件
```

---

## 6. 三条最值得记住的判断

1. **三个数字测的不是同一根轴**——190 是全后端目录，45 是 netplist 可见的高阶 descriptor 数，Zin IR 实际 op 数 > 45（lowering 之后还有一层）。把它们当成"输入 vs 输出"对比是误读。

2. **190 ∩ 45 ≠ 45**——Core ML 目录和 Zin IR 高阶 descriptor 是**两个交集非空的集合**，双方各有独占元素。Core ML 独占：CPU/GPU 专属 op；Zin IR 独占：SDPA / Sort / 点云 / 流式状态 / LUT 等"隐藏算子"。

3. **SDPA 不是 Core ML 融合出来的，是路径 B 独占触发**——Core ML → Zin IR 翻译器**不会**把 `MatMul → Mul → Softmax → MatMul` 自动折叠成 SDPA。想触发 SDPA，必须手写 netplist。这是 ch26 "隐藏算子"工程的全部价值所在。

---

## 7. 与其他章节的勾连

- **→ 第26章（Hidden layers & direct netplist authoring）**：本章把 ch26 路径图里"中间表示（Zin IR）"那一层拆成两层（高阶 descriptor + 低阶 op），把"45"这个数字的语义说清楚
- **→ 第24章（HAL and capability gates）**：路径 A 的"CPU/GPU 回退"发生在 router 阶段，对应第24章的 [算子校验流程](./arxiv-2606.22283-ch24-op-validation-flow.md) 里 `MinimumFamily` trait 检查失败
- **→ 第7章（Weights and compression）**：路径 A 的折叠/拆分跟第7章的 split-legalize（kernel-memory 上限触发的切层）是同一件事的两面
- **→ 第8章（Entitlement boundary）**：路径 B 触发"隐藏 descriptor"仍然要过两套 gate（`MinimumFamily` + HAL capability bytes），entitlement 不通就到不了芯片

---

*本文档基于 arXiv:2606.22283v1 (21 Jun 2026) 第26章 (pp. 165–169，§26.1–§26.3) 撰写。论文为逆向工程参考文档，所描述的私有接口未公开、未受支持、跨版本脆弱，仅供研究、测量和设备端实验使用。Core ML 仍是唯一受支持的发行路径。*
