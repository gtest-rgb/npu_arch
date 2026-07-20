# 第26章《Hidden layers & direct netplist authoring》精炼笔记

> 论文：*Apple Neural Engine: Architecture, Programming, and Performance*
> 作者：Spencer H. Bryngelson（Georgia Institute of Technology）
> 提交：2026-06-21，302 页，CC-BY 4.0
> 本章原文：pp. 165–169
>
> 这是 [深度解读版](./arxiv-2606.22283-ch26-hidden-layers-netplist.md) 的精炼版本，仅保留三件事：netplist 直写路径全景、45 个原生描述符里的隐藏算子、两套 gate 不一致与 Target 命名。

---

## 1. netplist 直写路径全景

### 1.1 两条路径对比

```
            Core ML 模型（.mlpackage）
                    │
                    │  框架层算子目录（≈190 种 op）
                    ▼
        ┌──────────────────────────────┐
        │  路径 A：Core ML → Zin IR     │  ← 翻译路由（sanctioned）
        │       翻译器（折叠/拆分）     │
        └──────────────┬───────────────┘
                       │
                       ▼
                中间表示（Zin IR）
        ┌──────────────────────────────┐
        │  原生硬件描述符集合（45 种）  │
        └──────────────┬───────────────┘
                       │
                       │  路径 B：手写 .espresso.net
                       │  ↓ 跳过翻译器，直接落在这里
                       ▼
                ANECompiler 后端 → .mlmodelc
                       │
                       ▼
                  AppleH11ANEInterface
                       │
                       ▼
                  ANE 固件
```

第26章的主角是**路径 B**——不经过 Core ML 翻译器、直接拿 netplist 喂给 Zin IR 的隧道。

### 1.2 netplist 是什么

| 属性 | 值 |
|---|---|
| 扩展名 | `.espresso.net`（"Espresso" 是 Apple 内部对 ANE 编译前端的老名字） |
| Schema 版本 | 1.0.10 |
| 物理格式 | Apple 属性表（plist），可 `plutil -p` 直接看 |
| 顶层 key | `Networks` / `ProcedureList` / `Units` / `Weights` |
| 解析器入口 | 每个 layer 一对：`ZinParse<Name>Unit` / `_ANEC<Name>LayerDescInitialize` / `_ANECValidate<Name>` |

### 1.3 一份 netplist 的骨架

```plist
Network {
    Name    = "my_handwritten_graph"
    Target  = "H13"              # M1 → H13
    Inputs  = [ "q", "k", "v" ]
    Outputs = [ "attn_out" ]

    Unit "attn" {
        Type       = "SDPA"            # 必须命中 45 个原生 descriptor 之一
        Bottom     = [ "q", "k", "v", "scale" ]   # Caffe 风格：Bottom=输入
        Top        = [ "attn_out" ]                # Top=输出
        Params     = { SubtractMax = true }
        OutputType = "Float16"
    }
}
```

- **Unit** = 图里的一个节点，对应 Zin IR 里的一个 operation
- **Type** 必须命中那 45 个原生描述符之一，否则对应 layer 的 `ZinParse<Name>Unit` 直接拒绝
- **Params** 字段名和编译器里的 `_ANEC<Name>LayerDescInitialize` 一一对应（每 layer 一套）
- **Weights** 直接内嵌压缩字节流（见第25章）

### 1.4 四步工作流

```
① 写 .espresso.net
     │
     ▼
② 通过 bridge node / tunneled-unit 路径把原生 Type 直接送后端
   （不经翻译，pass-through framework layer）
     │
     ▼
③ 编译器跑 validator + 分配引擎，产出 .mlmodelc
     │
     ▼
④ 加载 + 绑定张量 + 提交执行（第6章 direct dispatch 同款）
```

> **警告**：路径 B 是私有 API + 反编译得来，Apple 任意版本都可能改字段 / 符号 / schema。仅用于研究、测量、设备端实验。**Core ML 仍是唯一受支持的发行路径。**

---

## 2. 45 个原生描述符里的"隐藏算子"

整章最关键的部分。论文 §26.3 + Table 26.1 列出 7 类"隐藏算子族"——它们在 Core ML 的 ≈190 种 op 目录里**没有直接对应物**，但在编译器里有原生 descriptor。

### 2.1 总览表

| 算子族 | 代表 Type | Core ML 等价物 | M1 可达？ | 典型用途 |
|---|---|---|:-:|---|
| **融合注意力** | `SDPA` | **无**（必须拆成 4 个 op） | **✓** M1 起 | Transformer |
| **排序与选择** | top-k / sort / argmin-max | 无独立 sort | △ 部分 M1 拒 | NMS / beam search |
| **空间重排** | pixel-shuffle/unshuffle、space-and-batch | 部分 | ✓ M1 起 | 超分辨率 / ViT |
| **额外归一化** | range norm / LRN / gain-offset | 部分走 MPS | △ range norm M1 拒 | — |
| **几何与点云** | cross product / FPS / radius search | 无 | ✓ M1 起（cross） | 3D 感知 / AR |
| **纹理采样器** | resize / crop-resize / affine | 无原生 | **✗ M1 拒** | 几何变换 |
| **数据移动** | re-strided view / dynamic slice / tile | 部分 | △ dynamic-slice M1 拒 | DMA 精细控制 |
| **流式状态** | live state / ring-buffer / tensor-to-buffer | 外显成额外 I/O | △ circular mode M1 缺 | KV cache / RNN state |
| **可编程激活 LUT** | `kZinIrNonLinearCustomLUT` | 无 | ✗ netplist 不可达 | 分段折线激活 |

### 2.2 SDPA —— 隐藏算子的旗舰

- **Type 名**：`SDPA`（Scaled Dot-Product Attention）
- **Core ML 等价物**：**没有**。Core ML 必须把 attention 拆成 `MatMul → Mul(scale) → Softmax → MatMul` 四个 op
- **操作数契约**：4 或 5 个 Bottom —— `q, k, v, scale`，可选第 5 个**加性 mask 张量**
- **M1 起就原生跑**，不被纹理引擎门控（Table 26.1 明确点出）
- **意义**：把 4 个 op 压成 1 个 DMA 边界，省掉 3 次中间张量回写

### 2.3 SDPA 命门：`SubtractMax = true`

```plist
Unit "attn" {
    Type       = "SDPA"
    Bottom     = [ "q", "k", "v", "scale" ]
    Top        = [ "attn_out" ]
    Params     = {
        SubtractMax = true         # ← 必须开，否则 softmax 数值错
    }
    OutputType = "Float16"
}
```

**为什么是命门**：
- descriptor 构造函数**默认 `SubtractMax = false`**
- 对 softmax 是**数值错的**（fp16 溢出）
- 漏掉这个标志 = 手写 SDPA 最常见的错误

**因果掩码不是 Params 布尔位**：
- 它是**第 5 个加性 mask 操作数（数据）**
- 把对角线及以下设为 0、以上设为大负偏置，broadcast 到 heads
- `CausalMask` 之类的 Params 字段**在论文里不存在**

**scale 是操作数**：是 4 个 Bottom 之一（或 5 个中的第 4 个），**不是 Params 字面量**。论文里没有 `ScaleType` / `PerHead` 这类 Params 字段。

### 2.4 其他几族的 M1 细节

**排序与选择**
- top-k、sort、dynamic-slice 的 validator 都能调通，但 **M1 上 codegen 拒绝 sort 和 dynamic-slice**
- top-k 只在一段很窄的禁用参数带之外才接受
- whole-tensor argmin/argmax 走 `MinimumFamily`，**A15+ 才原生**

**纹理采样器**（M1 最大缺口）
- 被 capability byte `0x81d` 门控
- **A14+ 才接受，M1 直接报错** "affine transform is not supported on this architecture"

**可编程激活 LUT**（看得到够不着）
- 操作码 `kZinIrNonLinearCustomLUT`
- 33 节分段线性查找表（32 段）
- **raw custom-table path 实际上从 netplist 不可达**——unit parser 同时要求 saturation set + version-specific set，consistency check 又拒绝它们共存
- capability 是真的，但用户侧够不到；固件在 load 时从程序里 synthesize 出来
- 工程近似：用 `linear → relu → linear` 链（rectifier basis）在 fp16 精度内复现任意 knot table

---

## 3. 两套 gate 不一致 + Target 命名

### 3.1 翻译路由 vs 直写路由用两套不同的 gate

| 维度 | 路径 A（翻译路由 / Core ML） | 路径 B（直写路由 / netplist） |
|---|---|---|
| **gate 类型** | `MinimumFamily` 特性 trait | HAL capability bytes（位图） |
| **检查时机** | Core ML 转换阶段 | ANECompiler 后端 |
| **粒度** | op 类型级（"这个 op 需要 M2+"） | descriptor 字段级（"这个 Params 字段在 H14 不存在"） |
| **失败表现** | op 被静默退回 CPU | 编译直接报错 |
| **可观察性** | `coremlcompiler` 日志 | ANECompiler 私有日志 |

**含义**：
- 一个 op 在 Core ML 里"支持"，**不代表它走的就是 ANE**——可能被翻译器降级到 CPU
- 一个 Type 在 netplist 里能被 `_ANECValidate<Name>Layer` 通过，**也不代表 HAL capability bytes 答应**——可能在后端被踢掉
- **两套 gate 同时检查，任一不通过都到不了芯片**
- 反复出现的 motif：**"验证通过 ≠ 可达"**

### 3.2 Target 字段与芯片家族命名

`Target` 字段决定走哪个芯片家族，验证器拿它查该家族的 HAL capability bytes。

| 公开名 | 内部 H 名 | 出处 |
|---|---|---|
| A13 / **M1** | **H13** | 论文 Table 24.1、Listing 26.3 直接给出 |
| A14 / M2 | H14 | 外推（惯例） |
| A15 / M3 | H15 | 外推（惯例） |
| A16 / M4 | H16 | 外推（惯例） |
| A17 / **M5** | **H17** | 论文 §24.2 直接给出 |

`Target` 填错 = 验证器查错 HAL 表 = 编译失败。

---

## 4. 三个值得记住的判断

1. **45 ≷ 190 的反直觉**——Core ML 对外宣称 ≈190 种 op，但芯片真正认识的只有 45 个原生 descriptor。反过来，这 45 个里还**藏着 Core ML 不让你看到的"高阶积木"**（SDPA / Sort / 点云 / 流式状态 / LUT）。直写 netplist 可以触发它们。

2. **两套 gate 不一致是核心陷阱**——翻译路由用 `MinimumFamily` trait，直写路由用 HAL capability bytes，**两套互不买账**。"Core ML 支持" ≠ "ANE 原生" ≠ "直写可达"——这三个集合互不包含。

3. **SDPA 的 `SubtractMax = true` 是手写第一坑**——descriptor 默认 `false`，对 softmax 是数值错的。M1 起就原生跑 SDPA（不被纹理引擎门控），但漏掉 `SubtractMax` 等于白跑。

---

## 5. 速查图（一张图记住全章）

```
┌──────────────────────────────────────────────────────────────┐
│   Core ML 给你 ≈190 个 op = "翻译层视图"                      │
│                                                              │
│   ANECompiler 内部只有 45 个原生 descriptor                  │
│   但这 45 个里藏着 SDPA / Sort / 点云 / 状态流 / LUT 等       │
│   框架不让你看到的"高阶积木"                                  │
│                                                              │
│   你可以手写 .espresso.net 跳过翻译层，直接触发隐藏算子       │
│                                                              │
│   代价：                                                     │
│     · 必须自己处理两套 gate（MinimumFamily + HAL bytes）      │
│     · 必须为每个 Target 单独写一份（H13/H14/H15/...）         │
│     · 必须知道 SubtractMax 这种"命门级"细节                  │
│     · 必须接受跨版本脆弱、无支持、无文档                      │
│                                                              │
│   ★ 验证通过 ≠ 可达                                          │
│   ★ 编译通过 ≠ 跑得对                                        │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. 与其他章节的勾连

- **→ 第8章（Entitlement boundary）**：本章两套 gate 是第8章"entitlement gate + HAL capability"主题的细化
- **→ 第24章（HAL and capability gates）**：本章 §3 的 HAL capability bytes 是第24章 HAL 表的实例
- **→ 第24章专题（算子校验流程）**：本章两套 gate + "验证通过 ≠ 可达"是 [算子校验流程](./arxiv-2606.22283-ch24-op-validation-flow.md) 的具体表现
- **→ 第6章（Dispatching without Core ML）**：本章第④步 direct dispatch = 第6章"五步法"的后半段
- **→ 第25章（Compression internals）**：netplist 的 `Weights` 字段直接内嵌第25章讨论的压缩字节流

---

*本文档基于 arXiv:2606.22283v1 (21 Jun 2026) 第26章 (pp. 165–169) 撰写。论文为逆向工程参考文档，所描述的私有接口未公开、未受支持、跨版本脆弱，仅供研究、测量和设备端实验使用。Core ML 仍是唯一受支持的发行路径。*
