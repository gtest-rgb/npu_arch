# 第 26 章 隐藏层与直接 netplist 编写（Hidden Layers & Direct netplist Authoring）通俗解读

> 原论文：Spencer H. Bryngelson，《Apple Neural Engine: Architecture, Programming, and Performance》，arXiv:2606.22283，2026 年 6 月。
> 本章原文位于论文第 165–169 页。本文为基于原文的通俗中文解读，不是逐字翻译。

---

## 一、这一章到底在讲什么？

**一句话**：讲在 ANECompiler 内部，存在一份**只有 45 个原生描述符（descriptor）**的"硬件直通目录"，它和 Core ML 框架对外宣称的"约 190 种算子"不是一回事；以及——**理论上你完全可以绕过所有高层翻译，直接手写一份叫 netplist 的属性表，把图喂给这 45 个原生描述符**。

为什么这件事很重要？

- Core ML 给出的算子目录（≈190 种）经过翻译层，**才**变成芯片能直接吃的 45 种结构。
- 翻译层会"折叠"很多东西：例如把 attention 拆成 4 个小算子、把 reshape 融进 DMA。
- **但芯片里真正能直接识别的"高级算子"远比框架宣传的多**——SDPA、排序、点云重排都在里面。
- 这些"隐藏算子"在框架层是看不到的，但它们存在于编译器的中间表示（IR）里，可以通过手写 netplist 直接触发。

这一章拆开来讲三件事：

1. **netplist 是什么？长什么样？**（26.1 + 26.2）
2. **怎么绕过翻译层、直接触达那 45 个原生描述符？**（26.3）
3. **触达之后，能干哪些翻译层不让你干的事？**（26.4 + 26.5）

剩下的 26.6 是工程教训，26.7 是开发者影响。

---

## 二、先建立全景图：从 Core ML 到芯片的三条路径

```
            Core ML 模型（.mlpackage）
                    │
                    │  框架层算子目录（≈190 种 op）
                    ▼
        ┌──────────────────────────────┐
        │   Core ML → ANE IR 翻译器     │  ← 路径 A：翻译路由
        │   （把 190 种折叠 / 拆分）     │
        └──────────────────────────────┘
                    │
                    ▼
            中间表示（Zin IR）
        ┌──────────────────────────────┐
        │  原生硬件描述符集合（45 种）   │  ← 路径 B：直写路由
        └──────────────────────────────┘
                    │
                    │   手写 .espresso.net（netplist）
                    │   ↓ 跳过翻译器，直接落在这里
                    ▼
            ANECompiler 后端
                    │
                    ▼
              .mlmodelc 编译产物
                    │
                    ▼
               AppleH11ANEInterface
                    │
                    ▼
            ANE 固件（h13_ane_fw_styx_j5x.im4p 等）
```

第 26 章的主角就是**路径 B**：那条不经过 Core ML 翻译器、直接拿 netplist 喂给 Zin IR 的隧道。

---

## 三、netplist 是什么：一份手写的属性表

### 3.1 文件扩展名与 schema 版本

- 扩展名：`.espresso.net`（"Espresso"是 Apple 内部对 ANE 编译前端的老名字）
- Schema 版本（截至论文测量）：**1.0.10**（论文 §22 / §23 给出）
- 物理上就是一个 Apple 属性表（plist），可以用 `plutil -p` 直接打开看
- 解析器入口：每个 layer 有一对 `ZinParse<Name>Unit` / `_ANEC<Name>LayerDescInitialize` / `_ANECValidate<Name>` 例程（例如 `ZinParseSDPAUnit`），并没有一个统称的 `ZinParseUnit` 入口

### 3.2 顶层结构

一份 netplist 长这样（伪代码，字段名经过对齐；论文 Table 22.6 列出的顶层 key 是 `Networks`、`ProcedureList`、`Units`、`Weights` 等）：

```plist
Network {
    Name    = "my_handwritten_graph"
    Target  = "H13"              # M1 → H13，见 §六的命名规则
    Inputs  = [ "q", "k", "v" ]
    Outputs = [ "attn_out" ]

    Unit "attn" {
        Type       = "SDPA"
        Bottom     = [ "q", "k", "v", "scale" ]
        Top        = [ "attn_out" ]
        Params     = { SubtractMax = true }
        OutputType = "Float16"
    }

    Unit "proj" {
        Type       = "Convolution"
        Bottom     = [ "attn_out" ]
        Top        = [ "proj_out" ]
        Params     = {
            KernelSize = [ 1, 1 ]
            Stride     = [ 1, 1 ]
            Weights    = < ...bytes... >
        }
        OutputType = "Float16"
    }
}
```

几个关键点：

- **`Unit`** 就是图里的一个节点，对应 Zin IR 里的一个 Zin operation。
- **`Type`** 必须命中那 45 个原生描述符之一，否则对应 layer 的 `ZinParse<Name>Unit` 直接拒绝。
- **`Bottom` / `Top`** 是 Caffe 风格的命名张量连接（Bottom = 输入，Top = 输出）。
- **`Params`** 是该描述符的参数子表，字段名和编译器二进制里的 `_ANEC<Name>LayerDescInitialize` 一一对应（每 layer 一套）。
- **`Weights`** 直接内嵌压缩后的权重字节流（参见第 25 章的压缩格式）。

### 3.3 解析与验证的两道关

每一个 `Unit` 在被接受之前要经过一对 per-layer 例程（名字按 `Type` 实例化）：

1. **`_ANEC<Name>LayerDescInitialize`**：把 plist 里的 `Params` 字典填到一个 C 结构体里，做基本的类型检查。
2. **`_ANECValidate<Name>Layer`**：调对应描述符的验证器，检查字段齐全性、维度合理性、是否被当前 `Target` 支持。

注意：**验证器通过 ≠ 这一层在芯片上真能跑**。验证器只看"字段齐不齐、类型对不对"，不看"硬件 primitive 在不在"。后者要等 `Target` gate 在后端才检查。这是第 26 章反复强调的**"验证通过不等于可达"**原则。

---

## 四、四步工作流：手写一个 netplist 并跑起来

论文给出的工作流是：

```
   ① 写 .espresso.net
        │
        ▼
   ② 用 bridge node / tunneled-unit 路径把原生 Type 直接送后端
        │
        ▼
   ③ 编译器跑 validator + 分配引擎，产出 .mlmodelc
        │
        ▼
   ④ 加载 + 绑定张量 + 提交执行（第 6 章 direct dispatch 同款）
```

具体做法（论文 §26.3 + §26.5）：

- **第 ① 步**：按 §三的模板写 plist。`Target` 字段决定走哪个芯片家族（H13=M1，H17=M5；其余代际编号见 §六的命名规则）。
- **第 ② 步**：通过编译器的 "tunneled-unit / bridge node" 路径，把 `Type=<原生 descriptor 名>` 的 Unit **不经翻译**直接交给后端。论文 §26.2 强调这条路径"passes the descriptor through the framework layer untouched"，是直写能触达隐藏算子的关键。
- **第 ③ 步**：编译器跑该 layer 的 validator，分配引擎，落成编译产物（`.mlmodelc` bundle，含 `model.espresso.net` + 权重）。
- **第 ④ 步**：编译产物照第 6 章 "direct dispatch" 的同样方式加载、绑定输入张量、提交执行；目标芯片上的 compile-and-run 是**唯一**能确认"真的跑通"的判据。

**重要警告**（论文原文反复强调）：

- 这条路是**私有 API + 反编译得来**的，Apple 任何版本都可能改字段、改符号、改 schema。
- 仅用于研究、测量、设备端实验。
- **Core ML 仍然是唯一受支持的发行路径。**

---

## 五、那 45 个原生描述符里，藏着哪些"框架层看不到的算子"？

这是整章最关键的部分。论文 §26.3 + Table 26.1 列出 7 类"隐藏算子族"，它们在 Core ML 的 190 种 op 目录里**没有直接对应物**，但在编译器里有原生 descriptor：

### 5.1 融合注意力（Fused Attention / SDPA）

- **Type 名**：`SDPA`（Scaled Dot-Product Attention）
- **Core ML 等价物**：没有。Core ML 必须把 attention 拆成 `MatMul → Mul(scale) → Softmax → MatMul` 四个 op，因为它的 op 集合里没有 atom 能 lower 到这个单一 descriptor。
- **操作数契约**：4 或 5 个 Bottom —— `q, k, v, scale`，外加可选的第 5 个**加性 mask 张量**（因果掩码就是把这个 mask 留成"对角线及以下为 0、以上为大负偏置"的数据形态，**不是 Params 布尔位**）。
- **唯一 Params 关键字段**：`SubtractMax`（布尔）—— softmax 之前是否减去当前 row 的最大值。**论文强调：descriptor 构造函数默认是 `false`，对 softmax 是数值错的，因此手写 SDPA Unit 必须显式设为 `true`**。这是 SDPA 数值正确性的命门。
- **首次跑通的家族**：**M1 (H13) 起，不被纹理引擎门控**——这是 Table 26.1 明确点出的、隐藏算子里少数 M1 就能原生跑的。
- **意义**：原生 SDPA 把 4 个 op 压成 1 个 DMA 边界，省掉 3 次中间张量回写，带宽利用率显著提升。

### 5.2 排序与选择（Ranking and selection）

- **涵盖**：`top-k`、`sort`、`argument-minimum-and-maximum`、whole-tensor argument form
- 整数索引输出以 fp16 编码返回（在它们能产出的索引范围内是精确的）
- Core ML 公开算子里没有独立 sort（只能用 `TopK`、`Gather` 等组合模拟）
- **M1 上的门控细节**（Table 26.1）：top-k、sort、dynamic-slice 的 validator 都能调通，但 **code generator 在 M1 上拒绝 sort 和 dynamic-slice**，top-k 只在一段很窄的禁用参数带之外才接受。whole-tensor argmin/argmax 走 MinimumFamily，**A15+ 才原生**。
- 典型用途：NMS 后处理、beam search、top-1 采样

### 5.3 空间重排（Spatial rearrangement）

- **涵盖**：**pixel-shuffle / pixel-unshuffle** 一对、**channel-and-space** 一对、**space-and-batch** 一对
- 每对都用 per-axis integer factor 参数化；区别于别名，它们由 channel-ordering convention 区分
- M1 起原生支持
- 典型用途：超分辨率上采样（pixel-shuffle）、Vision Transformer patch embedding（space-and-batch）

### 5.4 额外的归一化（Additional normalizations）

- **涵盖**：range normalization（把张量映射到 min–max 区间）、local response normalization、per-channel affine gain-offset control（静态和 runtime-tensor 两种形式）
- **range normalization 是 arch-gated 的，M1 上拒绝**；gain-offset 在 M1 起就支持
- Core ML 公开 API 里这部分要么没有，要么要走 MPS

### 5.5 几何与点云（Geometry and point cloud）

- **涵盖**：template cross-correlation、three-vector cross product、furthest-point sampling、radius neighborhood search、stereo cost volume
- cross product 要求 interleave-one 布局和 fp16 操作数；template cross-correlation 的深度为 1
- M1 起对 cross / correlation 形式可用
- 典型用途：3D 感知、AR、自动驾驶、立体匹配

### 5.6 纹理采样器（Texture samplers）

- **涵盖**：作为硬件采样器的 resize、crop-and-resize、grid resample、affine spatial transform
- **门控**：被纹理引擎 capability byte (`0x81d`) 门控，**A14+ 才接受，M1 上拒绝**（报错信息形如 "affine transform is not supported on this architecture"）
- 这是隐藏算子里**最大的一族 M1 缺失项**

### 5.7 数据移动（Data movement）

- **涵盖**：re-strided input view、runtime-offset dynamic slice、tile、concatenate
- re-strided view 必须跟在一个 reshape 之后；dynamic slice 在 M1 上 validator 过但 codegen 拒
- 这些在框架层是"零成本"的元数据操作，但在 ANE 上**真的会占用 DMA 通道和周期**
- 直写 netplist 可以精确控制 DMA 的 tile 大小——这些是 Core ML 不会暴露的旋钮

### 5.8 流式状态（Streaming state）

- **涵盖**：live state、ring-buffer reader / writer、tensor-to-buffer movers
- ring-buffer writer 必须连到一个 live-state buffer；**circular mode 在 M1 上是 arch-gated 的，只能通过 shared resident buffer 间接达到**（见第 8 章对 counter-and-event engine 缺失的解释）
- 让 ANE 在不同 inference 之间保留一份片上状态（KV cache、RNN hidden state）
- Core ML 的 `mlmodel` 把状态外显成额外输入输出，但芯片里其实有专门的 state descriptor

### 5.9 可编程激活 LUT（Programmable Activation LUT）

- 操作码：`kZinIrNonLinearCustomLUT`
- 一张 **33 节分段线性查找表**（32 段），33 knot 值在固定 step 上均匀分布，32 个 inter-knot delta 紧跟在短 header 后
- 论文 §26.10 指出：**raw custom-table path 实际上从 netplist 不可达** —— unit parser 同时要求一个 saturation set 和一个 version-specific set，consistency check 又拒绝它们共存，因此**没有手写的表能同时满足**。capability 是真的，但用户侧够不到；固件在 load 时从程序里 synthesize 出来。
- 工程上的近似等价：任意 pointwise 函数都能写成 `linear → relu → linear` 链（rectifier basis），在 fp16 精度内复现任意 knot table。论文用 Gaussian bump 演示了这一点。
- 典型用途：量化感知推理里的分段折线、自定义门控函数、近似 GELU

---

## 六、芯片家族命名规则：M(n) → H(n+12)

`Target` 字段决定走哪个芯片家族。论文 ch24 / ch26 直接确认了两个锚点：**M1 = H13 = A13**（M1 ANE 与 A13 同代微架构）、**M5 = A17**。其它代际对应（M2→H14、M3→H15、M4→H16）是按 Apple 一贯的"A 系列 / H 系列 / M 系列同代对应"惯例**外推**的，论文 ch26 没逐项列出：

| 公开名 | 内部 H 名 | 出处 |
|---|---|---|
| A13 / M1 | H13 | 论文 Table 24.1、Listing 26.3 直接给出 |
| A14 / M2 | H14 | 外推（惯例） |
| A15 / M3 | H15 | 外推（惯例） |
| A16 / M4 | H16 | 外推（惯例） |
| A17 / M5 | H17 | 论文 §24.2 直接给出 |

`Target` 字段填错会直接编译失败——验证器会拿 `Target` 去查那个家族的 HAL capability bytes，看你要的 Type 在不在白名单里。

---

## 七、架构门控：两套 gate 怎么互相不买账

第 24 章讲过 HAL capability gates。第 26 章的补充关键点是：**翻译路由和直写路由用的是两套不同的门控**，互相不一致。

| 维度 | 路径 A（翻译路由 / Core ML） | 路径 B（直写路由 / netplist） |
|---|---|---|
| **gate 类型** | `MinimumFamily` 特性 trait | HAL capability bytes（位图） |
| **检查时机** | Core ML 转换阶段 | ANECompiler 后端 |
| **粒度** | op 类型级（如"这个 op 需要 M2+"） | descriptor 字段级（如"这个 Params 字段在 H14 不存在"） |
| **失败表现** | op 被静默退回 CPU | 编译直接报错 |
| **可观察性** | `coremlcompiler` 日志 | ANECompiler 私有日志 |

含义：

- 一个 op 在 Core ML 里"支持"，不代表它走的就是 ANE——可能被翻译器降级到 CPU。
- 一个 Type 在 netplist 里能被 `_ANECValidate<Name>Layer` 通过，也不代表 HAL capability bytes 答应——可能在后端被踢掉。
- **两套 gate 同时检查，任一不通过都到不了芯片**。

---

## 八、SDPA 完整示例：手写一遍原生 SDPA

把前面的知识串起来。下面是一份**最小可编译**的 SDPA Unit（论文 §26.4 Listing 26.1 的等价中文重述，plist 字段经过整理）：

```plist
Unit "attn" {
    Type       = "SDPA"
    Bottom     = [ "q", "k", "v", "scale" ]   # 可选第 5 个：加性 mask
    Top        = [ "attn_out" ]
    Params     = {
        SubtractMax = true                    # ← 必须开，否则 softmax 数值错
    }
    OutputType = "Float16"
}
```

要点（全部出自论文 §26.4 + Table 26.1）：

1. **Target 选 H13 即可**：论文 Table 26.1 明确标 "Fused attention... **M1, not texture-gated**"，§26.4 进一步说 "it runs on every family from the M1 onward"。SDPA 是隐藏算子里**少数 M1/H13 就原生跑**的，不是 H14+ 才支持。
2. **`SubtractMax = true`**：descriptor 构造函数默认是 `false`，对 softmax 是数值错的，**漏掉这个标志是手写 SDPA 最常见的错误**。
3. **因果掩码不是 Params 布尔位**：它是**第 5 个加性 mask 操作数（数据）**——把对角线及以下设为 0、以上设为大负偏置，broadcast 到 heads 上。`CausalMask` 之类的 Params 字段**在论文里不存在**。
4. **scale 是操作数**，是 4 个 Bottom 之一（或 5 个中的第 4 个），不是 Params 字面量。论文里没有 `ScaleType` / `PerHead` 这类 Params 字段。
5. validator 强约束：4 或 5 个 Bottom 必须存在；`key` 和 `value` 共享 shape；`q × kᵀ` 必须能 contract 到 `v`。

---

## 九、开发者影响：这条路径到底给谁用？

论文在 §26.7 明确把"谁应该用 netplist 直写"列了出来：

**适合用**：

- **性能测量 / 学术研究**：想测原生 SDPA 的真实延迟、想看 Sort descriptor 的寄存器占用。
- **逆向工程 / 安全研究**：研究 ANE 固件、命令协议、capability matrix。
- **片上原型**：在 Core ML 还没暴露某个 op 之前，先在 ANE 上验证可行性。
- **极限优化**：当一个模型的瓶颈在 DMA、而不是 MAC 时，手写 netplist 可以精确控制 tile。

**不适合用**：

- **任何要发布给最终用户的软件**：私有 API 跨版本脆弱，macOS/iOS 任意小版本都可能改符号。
- **跨芯片分发**：手写 netplist 必须为每个 Target 单独写一份（capability bytes 不一致）。
- **需要审核 / 公证的 App Store 应用**：用私有类（`AppleH11ANEInterface`）会直接被拒。

---

## 十、一句话总结

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   Core ML 给你 ≈190 个 op，那是"翻译层视图"。               │
│                                                              │
│   ANECompiler 内部只有 45 个原生 descriptor，                │
│   但这 45 个里藏着 SDPA、Sort、点云、状态流、LUT 等          │
│   框架不让你看到的"高阶积木"。                               │
│                                                              │
│   你可以手写 .espresso.net 跳过翻译层，                       │
│   直接触发这些隐藏算子——                                     │
│                                                              │
│   但代价是：                                                 │
│     • 必须自己处理两套 gate（MinimumFamily + HAL bytes）      │
│     • 必须为每个 Target 单独写一份                           │
│     • 必须知道 SubtractMax 这种"命门级"细节                  │
│     • 必须接受跨版本脆弱、无支持、无文档                      │
│                                                              │
│   验证通过 ≠ 可达。编译通过 ≠ 跑得对。                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

下一篇就该看 **第 27 章 Kernel driver and IOKit ABI**（私有类 `AppleH11ANEInterface` 在论文 p.177 出现的地方）和 **第 30 章 Host-to-firmware command protocol**——它们一起把本章的 netplist 串成一条完整的端到端直连路径。第 17 章（Model-design rules）是面向模型作者的规则集，与 command protocol 不是同一回事。
