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
- Schema 版本（截至论文测量）：**1.0.10**
- 物理上就是一个 Apple 属性表（plist），可以用 `plutil -p` 直接打开看
- 解析器入口符号：`ZinParseUnit`（在 ANECompiler 二进制里）

### 3.2 顶层结构

一份 netplist 长这样（伪代码，字段名经过对齐）：

```plist
SchemaVersion = "1.0.10"
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
- **`Type`** 必须命中那 45 个原生描述符之一，否则 `ZinParseUnit` 直接拒绝。
- **`Bottom` / `Top`** 是 Caffe 风格的命名张量连接（Bottom = 输入，Top = 输出）。
- **`Params`** 是该描述符的参数子表，字段名和编译器二进制里的 `_ANECLayerDescInitialize` 一一对应。
- **`Weights`** 直接内嵌压缩后的权重字节流（参见第 25 章的压缩格式）。

### 3.3 解析与验证的两道关

每一个 `Unit` 在被接受之前要经过两个函数：

1. **`_ANECLayerDescInitialize`**：把 plist 里的 `Params` 字典填到一个 C 结构体里，做基本的类型检查。
2. **`_ANECValidateLayer`**：根据 `Type` 调对应描述符的验证器，检查字段齐全性、维度合理性、是否被当前 `Target` 支持。

注意：**`_ANECValidateLayer` 通过 ≠ 这一层在芯片上真能跑**。验证器只看"字段齐不齐、类型对不对"，不看"硬件 primitive 在不在"。后者要等 `Target` gate 在后端才检查。这是第 26 章反复强调的**"验证通过不等于可达"**原则。

---

## 四、四步工作流：手写一个 netplist 并跑起来

论文给出的工作流是：

```
   ① 写 .espresso.net
        │
        ▼
   ② 用 ANECompiler 直接编译，跳过 Core ML 转换
        │
        ▼
   ③ 拿到 .mlmodelc，注入到 AppleH11ANEInterface 的私有 API
        │
        ▼
   ④ 提交 command buffer，观察输出 / 寄存器 / 中断
```

具体做法（论文 §26.3，decompile-derived）：

- **第 ① 步**：按 §三的模板写 plist。`Target` 字段决定走哪个芯片家族（H13=M1、H14=M2、H15=M3、H16=M4、H17=M5；A 系列 A13→H13，A14→H14，依此类推）。
- **第 ② 步**：调用 `ANECompiler` 的 `compile` 私有入口，传入 netplist 路径，跳过 Core ML 的 `espresso` 翻译层。
- **第 ③ 步**：编译产物是个 `.mlmodelc` bundle，里面是 `model.espresso.net` + `weights.bin`。用 `objc_getClass("AppleH11ANEInterface")` 拿到私有类，调 `-allocateNetworkWithDescriptor:...` 加载。
- **第 ④ 步**：构造 `ANEProgramRequest`，把输入张量地址填进去，`IOConnectCall` 提交。输出的 status code 在第 17 章 command protocol 里有完整表。

**重要警告**（论文原文反复强调）：

- 这条路是**私有 API + 反编译得来**的，Apple 任何版本都可能改字段、改符号、改 schema。
- 仅用于研究、测量、设备端实验。
- **Core ML 仍然是唯一受支持的发行路径。**

---

## 五、那 45 个原生描述符里，藏着哪些"框架层看不到的算子"？

这是整章最关键的部分。论文列出 6 类"隐藏算子"，它们在 Core ML 的 190 种 op 目录里**没有直接对应物**，但在 Zin IR 里有原生 Type：

### 5.1 融合注意力（Fused Attention / SDPA）

- **Type 名**：`SDPA`（Scaled Dot-Product Attention）
- **Core ML 等价物**：没有。Core ML 必须把 attention 拆成 `MatMul → Mul(scale) → Softmax → MatMul` 四个 op。
- **关键 Params**：
  - `SubtractMax`（布尔）：softmax 之前是否减去当前 row 的最大值。**论文强调：必须设为 `true`，否则 fp16 数值会在 head_dim 较大时溢出**。这是 SDPA 数值正确性的命门。
  - `CausalMask`（布尔）：是否做因果掩码（decoder 用）。
  - `ScaleType`：`Scalar` 或 `PerHead`（决定 scale 是一个数还是每头一个）。
- **意义**：原生 SDPA 把 4 个 op 压成 1 个 DMA 边界，省掉 3 次中间张量回写，带宽利用率显著提升。

### 5.2 排序（Sort）

- **Type 名**：`Sort`
- 支持升序 / 降序、支持沿指定轴。
- Core ML 公开算子里没有独立 sort（只能用 `TopK`、`Gather` 等组合模拟）。
- 典型用途：NMS 后处理、beam search、top-1 采样。

### 5.3 空间重排（Spatial Rearrange）

- **Type 名**：`SpatialRearrange`、`ImToCol`、`ColToIm`
- 把 im2col / col2im 这种展开做成原生 op，而不是让框架用 reshape+transpose 拼。
- 典型用途：传统卷积的 im2col 展开、vision transformer 的 patch embedding。

### 5.4 几何 / 点云（Geometry / Point Cloud）

- **Type 名**：`PointCloudProject`、`VoxelPool`
- 用于 3D 感知、AR、自动驾驶场景。
- Core ML 公开 API 里这部分要么没有，要么走 MPS。

### 5.5 数据移动（Data Movement）

- **Type 名**：`DMA`、`Concat`、`Split`、`Broadcast`、`Transpose`（带特殊 stride 模式）
- 这些在框架层是"零成本"的元数据操作，但在 ANE 上**真的会占用 DMA 通道和周期**。
- 直写 netplist 可以精确控制 DMA 的 tile 大小、双缓冲、流水线深度——这些是 Core ML 不会暴露的旋钮。

### 5.6 流式状态（Streaming State）

- **Type 名**：`StateRead`、`StateWrite`、`StateUpdate`
- 让 ANE 在不同 inference 之间保留一份片上状态（KV cache、RNN hidden state）。
- 这是 ANE 能做"有状态推理"的根基——Core ML 的 `mlmodel` 把状态外显成额外输入输出，但芯片里其实有专门的 state descriptor。

### 5.7 可编程激活 LUT（Programmable Activation LUT）

- **Type 名**：`NonLinear` 配合 `kZinIrNonLinearCustomLUT` 操作码
- 一张 **33 节分段线性查找表**（32 段）。
- 你可以在片上烧任意形状的分段线性激活函数（不是 ReLU/Swish 这种预定义的）。
- 典型用途：量化感知推理里的分段折线、自定义门控函数、近似 GELU。

---

## 六、芯片家族命名规则：M(n) → H(n+12)

论文在 §26.2 把 `Target` 字段的命名规则讲清楚了，这里直接抄一份表：

| 公开名 | 内部 H 名 | 说明 |
|---|---|---|
| A13 | H13 | iPhone 11 系列 |
| A14 | H14 | iPhone 12 系列 |
| A15 | H15 | iPhone 13 系列 |
| A16 | H16 | iPhone 14 Pro |
| A17 | H17 | iPhone 15 Pro |
| M1 | H13 | 与 A13 同代 ANE 微架构 |
| M2 | H14 | 与 A14 同代 |
| M3 | H15 | 与 A15 同代 |
| M4 | H16 | 与 A16 同代 |
| M5 | H17 | 与 A17 同代 |

**规则**：`H(n) ↔ M(n−12) ↔ A(n)`（n ≥ 13）。`Target` 字段填错会直接编译失败——验证器会拿 `Target` 去查那个家族的 HAL capability bytes，看你要的 Type 在不在白名单里。

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
- 一个 Type 在 netplist 里能被 `_ANECValidateLayer` 通过，也不代表 HAL capability bytes 答应——可能在后端被踢掉。
- **两套 gate 同时检查，任一不通过都到不了芯片**。

---

## 八、SDPA 完整示例：手写一遍原生 SDPA

把前面的知识串起来。下面是一份**最小可编译**的 SDPA netplist（论文 §26.4 风格，字段名经过整理）：

```plist
SchemaVersion = "1.0.10"
Network {
    Name    = "minimal_sdpa"
    Target  = "H14"                    # M2 / A14 起支持原生 SDPA
    Inputs  = [ "q", "k", "v" ]
    Outputs = [ "attn_out" ]

    Unit "scale_const" {
        Type       = "Constant"
        Top        = [ "scale" ]
        Params     = { Value = 0.125 }    # 1/sqrt(head_dim)
        OutputType = "Float16"
    }

    Unit "attn" {
        Type       = "SDPA"
        Bottom     = [ "q", "k", "v", "scale" ]
        Top        = [ "attn_out" ]
        Params     = {
            SubtractMax = true            # ← 必须开，否则 fp16 溢出
            CausalMask  = true            # decoder 用
            ScaleType   = "Scalar"
        }
        OutputType = "Float16"
    }
}
```

要点：

1. `Target = "H14"`：论文测量确认原生 SDPA 在 H14（M2/A14）及以上才进入 HAL capability 白名单。H13（M1/A13）写也会过 `_ANECValidateLayer`，但后端会被 capability bytes 拒掉。
2. `SubtractMax = true`：fp16 最大值约 65504，head_dim=128 时 softmax 分母会爆。SubtractMax 把每行减去最大值，等价于数值稳定的 softmax。**漏掉这个标志是手写 SDPA 最常见的错误**。
3. `scale` 作为单独的 `Constant` 单元产出——因为 `ScaleType = "Scalar"` 要求 scale 是一个张量输入，而不是 Params 里的字面量。

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

下一章（第 27 章）会讲 netplist 之上的更高一层抽象——直接和 AppleH11ANEInterface 的 command buffer 打交道的"协议层"，把第 17 章的 command protocol 和本章的 netplist 串成一个完整的端到端直连路径。
