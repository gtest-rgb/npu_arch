# arXiv:2606.22283 第8章《Entitlement boundary》精炼笔记

> 论文：*Apple Neural Engine: Architecture, Programming, and Performance*
> 作者：Spencer H. Bryngelson（Georgia Institute of Technology）
> 提交：2026-06-21，302 页，CC-BY 4.0
> DOI: `10.48550/arXiv.2606.22283`
> 本章原文：pp. 47–50
>
> 这是 [深度解读版](./arxiv-2606.22283-ch8-entitlement-boundary.md) 的精炼版本，仅保留三件事：通路图、能力表、管控点。

---

## 1. 直连路径的通路图

直连路径（direct route）从用户进程发起，经过**唯一能签名的系统守护进程**，再过内核驱动双校验，最终落在引擎上。**任何自编译或手写的程序字节都无法绕过守护进程**——这是整条通路的根。

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          直连路径（Direct Route）                           │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌───────────────────────┐                                               │
│  │  用户进程（Client）     │  ①  Author                                   │
│  │  · 不持任何 entitlement │  ── 以 IR（中间表示）写网络，不构造可加载二进制   │
│  │  · 不接触 framework     │     （自签二进制永远过不了 ④ 的内核加载校验）      │
│  └───────────┬───────────┘                                               │
│              │  ②  Delegate signing                                       │
│              │     把 IR 交给守护进程                                       │
│              ▼                                                            │
│  ┌───────────────────────┐                                               │
│  │  系统守护进程（Daemon） │  ★ 唯一能 compile + sign 的进程                  │
│  │  · 接收 IR              │     就地编译，就地签名                            │
│  │  · 编译器 = 可达性边界   │     返回带 trustcache trust 的程序                │
│  └───────────┬───────────┘                                               │
│              │  返回签名后的 program                                        │
│              ▼                                                            │
│  ┌───────────────────────┐                                               │
│  │  用户进程（Client）     │  ③  Drive                                     │
│  │  · 直接驱动签名程序     │  ── 通过 kernel interface 提交                  │
│  └───────────┬───────────┘                                               │
│              │                                                            │
│              ▼                                                            │
│  ┌───────────────────────┐                                               │
│  │  内核驱动（Kernel）     │  ④  Load-time 双校验                            │
│  │  · corecrypto(prog)    │     ── 程序字节签名校验                          │
│  │  · trustcache(vnode)   │     ── backing file 信任校验                    │
│  │  · 失败 → 0xe00002e2   │                                               │
│  └───────────┬───────────┘                                               │
│              │  双校验通过                                                  │
│              ▼                                                            │
│  ┌───────────────────────┐                                               │
│  │  ANE 引擎（硬件）       │  ⑤  执行                                       │
│  └───────────────────────┘                                               │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

★ 关键不变量：
  · 客户端永远拿不到"未签名的可加载二进制"——签名权被守护进程独占
  · 因此 "可达面 = 守护进程编译器愿意接受的一切"，而非"引擎原则上能跑的一切"
  · 攻击面从"任意 ANE 字节码"压缩到"IR grammar + 守护进程编译器漏洞"
```

**和 sanctioned（Core ML）路径的对比**：sanctioned 路径在用户进程和守护进程之间多了 framework 层（segmenter / model container / loader-tier features），但**到达的是同一块引擎**。Entitlement 多出来的是 loader 便利性，不是另一块引擎。

---

## 2. 直连路径的能力表

### 2.1 直连可达（无需任何 entitlement）

下列 op 在直连路径上完全可用，仅受第4章给出的 per-chip 形状限制约束：

| 类别 | 直连路径可触达的 op |
|---|---|
| 卷积 | 2D 卷积及其转置 |
| 矩阵 | 矩阵乘（matmul） |
| Attention | Fused attention（SDPA） |
| 归一化 | Normalization 族（BN / LN / GN 等） |
| 激活 | Activation table |
| 池化 | Pooling |
| 逐元素 | Elementwise arithmetic |
| 数据搬运 | Data-movement ops |
| 压缩权重 | 支持的 family 上流式加载，不支持的 family 重建回 fp16 |

**这覆盖了设备端感知网络、数值计算网络所需的全部原语。**

### 2.2 直连被门控（即使 sanctioned 路径下也不在引擎上跑）

| 特性 | 门控层 | Direct 表现 | Sanctioned 表现 | 是否 M1 上引擎可达 |
|---|---|---|---|:-:|
| **3D 卷积** | backend lowering | frontend 接受（要求 static weight），lowering 在每个 device mask 上失败 | segmenter 直接放到 CPU/GPU | ✗ |
| **Native stateful types**（state / ring buffer） | **硬件 primitive 缺失**（最强） | 编译失败；counter-and-event engine 在 M1 descriptor 里被 stub | framework 接 wiring，引擎仍缺 primitive | ✗ |
| **bf16 program I/O** | serialization dtype enum | `Unsupported function output dtype bf16` | framework 接受 bf16 数组，**进引擎前 cast 成 fp16** | ✗ |
| **Flexible / symbolic shapes** | runtime path（不是 compiler） | `?` 能 parse、能 bind，lowering 失败 | 预编译一组 fixed shape + 调度最接近的 + CPU 处理 dynamic remainder | ✗ |
| **Direct image-format input**（`pixel_buffer_to_tensor`） | backend lowering | IR 能 parse、能 type-check，backend 不 lower | framework 提供 image-input descriptor + surface setup | ✗（直连替代：on-engine int→fp16 dequant） |

**Program-I/O dtype 11 个 code**：`int4, uint8, int8, fp16, fp32, int16, uint16, int32, uint32, int64, uint64`（**bf16 不在内**）。

**共同模式**：每个被门控的特性都是**在上一层被接受、在下一层被拒绝**。

---

## 3. 直连路径的管控点

管控点按"离引擎远近"从远到近排列，**越靠近硬件的管控越不可绕过**：

```
┌──────────────────────────────────────────────────────────────────────┐
│   管控点                 │  位置              │  失败码 / 表现          │  强度
├─────────────────────────┼───────────────────┼────────────────────────┼──────────
│  ①  Entitlement gate    │  用户进程入口       │  0xe00002c7            │  最弱
│                          │                   │  (kIOReturnUnsupported) │
│  ②  守护进程编译器接受    │  Daemon compile   │  IR parse / type-check  │  ↑
│     （可达性的真实边界）  │                   │  通过，lowering 拒绝     │
│  ③  Backend lowering     │  Daemon backend   │  "Not implemented on    │  ↑
│                          │                   │  any of the backends"   │
│  ④  Load-time 双校验      │  Kernel driver    │  0xe00002e2            │  ↑
│     corecrypto + trustcache                  │  （签名 / vnode）        │
│  ⑤  硬件 primitive       │  Silicon          │  register setter assert │  最强
│                          │                   │  unsupported            │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.1 五个管控点逐一说明

**①  Entitlement gate（最弱）**
- 位置：用户进程调用 inference-tier 高阶特性时
- 失败码：`0xe00002c7`（`kIOReturnUnsupported`）
- 直连路径**根本不触碰**这里——它不申请 entitlement
- 注意：四项门控特性**全都失败在更上游**（②③⑤），所以 host 侧 entitlement 对它们**一个都撼动不了**

**②  守护进程编译器接受（可达性的真实边界）**
- 位置：系统守护进程的编译器前端
- 论文原文："reachable surface = everything the system daemon's compiler will accept"
- **这才是 direct route 的真正边界**，而不是 entitlement
- IR grammar + 编译器漏洞 = 整个 direct route 的攻击面

**③  Backend lowering**
- 位置：编译器后端代码生成
- 表现：3D conv / symbolic shape / `pixel_buffer_to_tensor` 都死在这里
- "Not implemented on any of the specified backends"
- frontend 可以识别，但没有任何 backend 为它生成代码

**④  Load-time 双校验（整个 access model 的根）**
- 位置：ANE 内核驱动，program bytes 提交到 firmware 之前
- 两道校验：
  - **corecrypto 签名校验**：对 program bytes 做签名验证
  - **Trustcache 校验**：对 backing file 的 vnode 做 trust 检查
- 失败码：`0xe00002e2`
- **唯一能通过的程序 = 守护进程 compile+sign 的程序**
- 这就是为什么 direct route 走三步法（Author → Delegate signing → Drive）—— 必须借守护进程的手签名

**⑤  硬件 primitive（最强，无法被软件绕开）**
- 位置：硅片本身
- 实例：M1 上的 counter-and-event DMA engine
  - 编译器保留了 operation class（命名了 ring-buffer reader/writer、counter-and-event DMA opcodes）
  - 但每个 register setter 的函数体都是 `assert(unsupported on this architecture)`
  - 即源/目的基址、shape、stride、原子操作、counter-mode、wait-event 地址和值——**全都 assert**
- **关键**：硅片路径里根本没有这个引擎，**任何 entitlement、property、compiler option 都无法合成它**
- 后续世代才加入，所以这是 per-family 特性，不是 M1 永久缺失

### 3.2 两个返回码——必须分清

| 码 | 来源 | 含义 |
|---|---|---|
| `0xe00002e2` | 签名 / trustcache 检查（管控点④） | 程序不被允许加载 |
| `0xe00002c7` | entitlement gate（管控点①） | 高阶 inference-tier 特性的 entitlement 缺失 |

**四项门控特性全都失败在更上游**（②③⑤），所以 host 侧 entitlement 对它们一个都撼动不了。

---

## 4. 三个值得记住的判断

1. **"Entitled ≠ reachable"**。授权边界不是"能用 vs 不能用"的分界，而是"loader 便利性 vs 自管编译"的分界。两条路径到达的是**同一块引擎**。

2. **门控强度有层级**：entitlement < 编译器拒绝 < lowering 未实现 < 签名校验 < 硬件 primitive 缺失。**最底层（硬件缺失）无法被软件绕开**——M1 的 native stateful types 就是反例。

3. **Direct route 的真正边界不在 entitlement，而在守护进程编译器**。Apple 的安全模型把"什么程序能跑"收敛到"守护进程愿意编译并签名什么"，把攻击面压缩到 IR grammar + 编译器漏洞。

---

## 5. 与其他章节的勾连

- **→ 第4章（Capability surface）**：§4.4 "Attested is not reachable" 被 §8.2 的 counter-and-event engine 实例化
- **→ 第6章（Dispatching without Core ML）**：本章 §8.6 的三步法是第6章"五步法"的信任根
- **→ 第7章（Weights and compression）**：压缩权重走 direct route，**不**触碰本章四项门控
- **→ 第24章（HAL and capability gates）**：HAL capability byte 是本章 entitlement 边界的硬件侧对应物
- **→ 第27章（Kernel driver and IOKit ABI）**：`0xe00002e2` / `0xe00002c7` 的实际触发位置
- **→ 第32章（Security and isolation）**：trustcache + corecrypto 是整个 isolation 模型的支点

---

*本文档基于 arXiv:2606.22283v1 (21 Jun 2026) 第8章 (pp. 47–50) 撰写。论文为逆向工程参考文档，direct route 未被 Apple 官方支持、跨版本脆弱，仅适用于研究、测量和设备端实验，不应用于发布软件。Core ML 仍是唯一受支持的发行路径。*
