# arXiv:2606.22283 第8章《Entitlement boundary》深度解读

> 论文：*Apple Neural Engine: Architecture, Programming, and Performance*  
> 作者：Spencer H. Bryngelson  
> 单位：Georgia Institute of Technology  
> 提交：2026-06-21，302 页，12 图  
> 许可：CC-BY 4.0  
> DOI: `10.48550/arXiv.2606.22283`

---

## 0. 章节定位

第8章位于论文 **Part II "Reaching the ANE"** 的收尾位置，紧接第7章《Weights and compression》、之后是 Part III 的第9章《Roofline》。

它回答一个核心问题：

> **绕过 Core ML 的"直连路径" (direct route) 究竟能到达 ANE 的哪些功能？在哪里被挡住？为什么？**

---

## 1. 核心论点 (SUMMARY)

作者用一句话概括全章逻辑：

> **直连路径能触达引擎所有"有用的计算"；只有四项特性被挡在 framework loader 或 entitlement 之后。** 这四项即使在 sanctioned（官方 Core ML）路径下，也无法在 M1 上跑在引擎上。

这把读者的直觉纠正了过来：授权边界 **不是 "能用 vs 不能用" 的分界**，而是 "loader 便利性 vs 自管编译" 的分界，**不会带来不同的引擎**。

---

## 2. 两条路径的结构对比

| 维度 | Direct route（直连） | Sanctioned route（Core ML） |
|---|---|---|
| 经过 framework | 否 | 是 |
| Placement planner | 无 | 有 segmenter，把模型切分到 CPU/GPU/ANE |
| Model container | 无 | 有（含 flexible-shape、stateful wiring） |
| Loader-tier 特性 | 无 | 有 |

**关键点**：直连路径 **直接走 execution runtime**，省掉了 Core ML 的 segmenter / model container / loader 特性层，但到达的是 **同一块引擎**。

---

## 3. 直连路径能做什么 (§8.1)

下列操作在直连路径上 **完全可用、无需任何 entitlement**，只受第4章给出的 per-chip 形状限制约束：

- 2D 卷积及其转置
- 矩阵乘
- Fused attention
- Normalization 族
- Activation table
- Pooling
- Elementwise arithmetic
- Data-movement 操作
- 压缩权重流（支持的 family 上流式加载，不支持的 family 重建回 fp16）

**这覆盖了设备端感知网络、数值计算网络所需的全部原语。**

---

## 4. 四项被门控的特性 (§8.2, Table 8.1)

这是全章最关键的一张表。**每项被门控的特性都通过更早的层，在 "执行工作的那一层" 失败** —— 失败点不同，门控机制不同。

### (a) 3D 卷积
- **门控层**：backend lowering（代码生成后端）
- **Direct 表现**：frontend 识别（要求 static weight），但 lowering 在 **每个 device mask** 上都以 "Not implemented" 失败
- **Framework 表现**：segmenter 直接把它放到 **CPU 或 GPU**，不上引擎
- **结论**：M1 上 **没有任何调用方能把 3D 卷积跑在引擎上**

### (b) Native stateful types (state / ring buffer)
- **门控层**：**硬件 primitive 缺失**（最强门控）
- 编译器保留了 operation class（命名了 ring-buffer reader/writer、counter-and-event DMA opcodes），但 counter-and-event engine 在 M1 硬件描述符里 **被 stub 掉了**
- 每个 register setter（源/目的基址、shape、stride、原子操作、counter-mode、wait-event 地址和值）的函数体都是 `assert(unsupported on this architecture)`
- **关键结论**：这比 entitlement 更强 —— **硅片路径里根本没有这个引擎**，任何 entitlement、property、compiler option 都无法合成它
- **替代方案**：M1 上用 shared buffer + cross-dispatch residency 实现等效 resident cache（文档化的 zero-copy 路径）
- **注**：counter-and-event engine 在 **后续世代** 才加入，所以这是 per-family 特性，不是 M1 的

### (c) bf16 程序输入/输出
- **门控层**：serialization 层 + datapath
- `ANECIRDataType` 枚举共 11 个 code：int4/uint8/int8/fp16/fp32/int16/uint16/int32/uint32/int64/uint64，**没有 bf16**
- Direct：声明 bf16 输入/输出 → 编译时 unsupported-dtype 拒绝
- Framework：接受 bf16 数组，但 **在进入引擎前先 cast 成 fp16**
- **结论**：sanctioned 路径下引擎也 **不跑 bf16 datapath**；第3章已说明 fp16 datapath 已经保留了 bf16 本应带来的累加精度

### (d) Flexible / symbolic shapes
- **门控层**：runtime path（不是 compiler！）
- 编译器有完整的 symbolic-shape 系统，master gate 在 M1 上是 **on**
- Direct：symbolic dimension `?` 能 parse、能 bind，但 **在 lowering 阶段失败**
- Framework：靠 "预编译一组 fixed shape + 调度最接近的一个 + CPU 上处理 dynamic remainder" 实现
- 本质：**bucketed fixed-shape specialization**，不是一个 symbolic 程序在引擎上跑
- Direct 路径的等价做法：padding 到 fixed maximum 或编译一组 length bucket，**compile cache 让重复 shape 几乎免费**

### 共同模式
每项特性都是 **在某一层被接受、在下一层被拒绝**。作者用 Listing 8.1 给出三个最小例子：

- 3D conv → frontend OK, lowering fails
- bf16 output → "Unsupported function output dtype bf16"
- `?` → parses, lowering "Not implemented"

---

## 5. 第五个边界：Image-input (§8.3)

作者单独拎出来，因为它 **形态更尖锐**：

- **目标**：把 camera/video surface 用原生 four-character-code pixel format 直接送进引擎，**免掉 host 侧到 fp16 的转换**
- **关键 op**：`pixel_buffer_to_tensor`
- **状态**：IR parser 能识别，`pixel_buffer` surface type、`FMT_*` 格式 token、grammar、format enum、type rule **全部可达且可解**
- **失败点仍是 lowering** —— direct 路径上 IR 能 parse、能 type-check、能走到 engine backend，但 **就在 backend 不 lower**
- Framework 路径提供 image-input descriptor 和 lowering 所需的 surface setup
- **直连替代**：把 byte 输入用 **on-engine integer → fp16 dequant** 替代 host-side 转换，绕开不支持 op，仍然消除了 host-side conversion

---

## 6. 边界的工程含义 (§8.4)

> "The choice between the routes is a choice of **loader and convenience**, not of **reachable compute**."

- Vision、encoders、on-device numerics、支持 op set 的 training → 全部基于 direct route 能编译调度的 op，**没有一项触碰门控集**
- Entitled 路径只多了一个 **有界清单**：segmenter + model container + loader-tier feature set
- **不会多出一块不同的引擎**：四项门控特性在 M1 上两条路径下 **要么同样缺失，要么被直连构造等价替代**

这是作者反复强调的 takeaway —— **不要把 entitlement 想成能力开关**，它只是 Apple 软件 stack 的 loader 差异。

---

## 7. 最底层硬限制：加载时签名校验 (§8.5)

> 全章最重要、也是整个 access model 的根：

**任何手工拼装或自编译的程序都无法被加载上引擎。** 机制：

1. **corecrypto 签名校验**：kernel driver 对 program bytes 做签名验证
2. **Trustcache 校验**：对 backing file 的 vnode 做 trustcache trust 检查
3. **失败返回**：`0xe00002e2`

**唯一能被加载的程序** 是系统 daemon **就地编译并签名** 的。这正是 direct route 的设计原因 —— 它不构建可加载二进制，而是：

- 作者把网络以 IR 形式表达
- IR 交给 daemon
- daemon 编译 + 签名
- 调用方驱动返回的签名程序

> **Reachable surface = daemon 编译器能接受的一切，而不是引擎原则上能跑的一切。**

### 7.1 详解：Trustcache trust 是什么

论文 §8.5 提到的 "trustcache check on the backing file's vnode" 是整个 access model 的支点，但论文本身没展开讲。这里补充背景。

**Trustcache 是什么**

Trustcache 是 Apple 平台代码签名体系里的核心信任机制，本质上是**一份内核常驻的"已知良好二进制"哈希白名单**。每个 trustcache 是一张 `CDHash → 信任` 的查找表：

- **CDHash** = Code Directory Hash，是整个二进制代码目录的 SHA-256，唯一标识"这份 Mach-O + 它的签名结构"
- 表里的每一项代表："这个哈希对应的二进制是 Apple 预先批准过的，加载时可跳过完整证书链验证"
- 表存放在内核内存里，由 boot chain（LLB → iBoot → kernel）在启动时装载，**用户进程不可改**

通俗类比：边防有一本"已审核护照号码本"，护照号在本本里的直接放行，不再问签证细节；不在本本里的才走完整审查。

**为什么存在这个机制**

完整代码签名校验（证书链 → 时间戳 → 撤销列表 → notarization）开销不小，Apple 系统里有几千个系统二进制频繁加载。所以把"已知良好"的哈希预先放进 trustcache，命中就快路径放行；用户安装的 app 不在 trustcache 里，走正常 Developer ID 签名校验。

iOS 越狱研究里 "trustcache injection" 是经典攻击目标——**把自己的哈希塞进 trustcache，就获得了"系统级信任"**。

**为什么 ANE 加载需要两道校验**

```
                  program bytes         backing file (vnode)
                       │                       │
                       ▼                       ▼
              ┌─────────────────┐    ┌────────────────────┐
              │ corecrypto 签名 │    │ trustcache 校验     │
              │ 校验            │    │ (CDHash ∈ cache?)  │
              └────────┬────────┘    └─────────┬──────────┘
                       │                       │
                       └───────────┬───────────┘
                                   ▼
                          两道都过 → 加载入引擎
                          任一失败 → 0xe00002e2
```

- **corecrypto 校验**：对 program bytes 做签名验证（字节层面的真实性）
- **trustcache 校验**：对 backing file 的 vnode（文件系统里的那个文件对象）做哈希检查（文件层面的可信度）

只做 corecrypto 还不够，因为：

- corecrypto 验的是"程序字节有没有被篡改"——保证真实性
- 但无法回答"这份程序是不是 Apple 批准过、能不能放进 ANE 跑"——这是**策略**问题，不是密码学问题

trustcache 把策略钉死在内核里：**只有哈希进了 trustcache 的程序文件，才有资格进 ANE**。而 trustcache 是 boot chain 装载的、用户进程改不了，所以这成了"硬件级信任根"的延伸。

**为什么签名权被守护进程独占**

把上面串起来就懂了为什么论文说"**唯一能加载的程序是系统守护进程编译并签名的**"：

1. 守护进程在系统启动时就跑着，它的可执行文件**已经在 trustcache 里**
2. 守护进程运行时**动态编译**用户提交的 IR，把编译产物作为临时文件写盘
3. **这个临时文件的 vnode 被加进 trustcache**（通过特权路径，只有系统进程能做）
4. 客户端通过 kernel interface 提交同一份文件的 vnode 去加载 → 双校验通过

普通用户进程**没有把哈希加进 trustcache 的能力**，所以自签二进制永远过不了第二道校验，必然返回 `0xe00002e2`。

**和 entitlement / sandbox 的层次区别**

| 机制 | 检查对象 | 谁能修改 | 在 ANE 流程里位置 |
|---|---|---|---|
| Entitlement | 进程签名里的能力清单 | 开发者（构建时）+ Apple（审批） | 最上游，用户进程入口 |
| Sandbox | 进程能否访问资源 | 系统策略 | 进程运行时 |
| **Trustcache** | **二进制哈希白名单** | **boot chain，用户进程不可改** | **ANE 程序加载时（最关键一道）** |
| Code signing | 证书链有效性 | 开发者（签名） | 进程启动 / 库加载时 |

trustcache 比 entitlement 更底层、更硬：**entitlement 是"允许做什么"的策略，trustcache 是"承认谁存在"的事实**。Apple 通过把"ANE 程序哈希塞进 trustcache"这件事独占给守护进程，把整个 ANE 的可达面收敛到了守护进程编译器。

> **一句话**：trustcache trust = "这个文件的哈希在内核的预批准白名单里吗？" 是 ANE 加载流程里**比 corecrypto 更策略化、也更难绕过**的一道关卡，也是守护进程能独占 ANE 签名权的物理基础。

---

## 8. Unentitled dispatch 三步法 (§8.6)

把签名约束工程化为三步协议：

1. **Author**：以 intermediate language（不是 loadable binary）写网络
2. **Delegate signing**：交给 daemon（**唯一能签名的进程**），它编译 + 就地签名，返回带 trustcache trust 的程序
3. **Drive**：客户端直接通过 kernel interface 驱动签名程序 —— corecrypto + trustcache 双校验通过，程序加载入引擎

这三步是 direct route 能在 "无 entitlement" 下工作的 **全部秘密**。

---

## 9. 参考表：边界常量 (§8.7, Table 8.2)

M1/H13 上的硬常量清单：

| 常量 | 值 |
|---|---|
| 程序加载拒绝码 | `0xe00002e2` |
| Program-I/O dtype codes | 11 个（int4, uint8, int8, fp16, fp32, int16, uint16, int32, uint32, int64, uint64） |
| bf16 program I/O | **不在 11 codes 中**，编译拒绝 |
| 3D 卷积 | 任何 device mask 都无 backend lowering |
| Native stateful types | counter-and-event DMA engine 在 M1 descriptor 中被 stub |
| Flexible / symbolic shapes | parse + bind 通过，direct runtime 上 lowering 失败 |
| 加载时签名校验 | corecrypto(program bytes) + trustcache(vnode) |
| Entitlement 拒绝码 | `0xe00002c7` (`kIOReturnUnsupported`) |
| Counter-and-event engine | 每个 register setter 都 assert unsupported |
| 直接图像格式输入 | `pixel_buffer_to_tensor` 可 parse，direct 路径不 lower |

### 关键区分：两个返回码

| 码 | 来源 | 含义 |
|---|---|---|
| `0xe00002e2` | 签名 / trustcache 检查 | 程序不被允许加载 |
| `0xe00002c7` | entitlement gate | 高阶 inference-tier 特性的 entitlement 缺失 |

四项门控特性 **全都失败在更上游**（compiler 或 firmware descriptor），所以 **host 侧 entitlement 对它们一个都撼动不了**。

---

## 10. 三个值得记住的判断

1. **"Attested is not reachable"** —— 硬件 capability table 里 attested 的特性未必可达（counter-and-event engine 被 stub 是最典型的反例）。这个 motif 在 §4.4 和 §24.8 反复出现。

2. **门控强度有层级**：entitlement < 编译器拒绝 < lowering 未实现 < 硬件 primitive 缺失。**最底层（硬件缺失）无法被软件绕开** —— 这是 M1 native stateful types 失败的根本原因。

3. **Direct route 的真正边界不在 entitlement，而在 daemon 编译器**。Apple 的安全模型把 "什么程序能跑" 收敛到 "daemon 愿意编译并签名什么"，把攻击面从 "任意 ANE 字节码" 压缩到 "IR grammar + daemon 编译器漏洞"。

---

## 11. 与其他章节的勾连

- **→ 第4章 (Capability surface)**：§4.4 "Attested is not reachable" 直接被 §8.2 的 counter-and-event engine 实例化
- **→ 第3章 (Numerics)**：bf16 门控与 fp16 datapath 的累加精度互为因果
- **→ 第6章 (Dispatching without Core ML)**：§8.6 是第6章 "五步法" 的信任根
- **→ 第24章 (HAL and capability gates)**：capability-byte gate 是本章 entitlement 边界的硬件侧对应物
- **→ 第27章 (Kernel driver and IOKit ABI)**：`0xe00002e2` / `0xe00002c7` 的实际触发位置
- **→ 第32章 (Security and isolation)**：trustcache + corecrypto 是整个 isolation 模型的支点

---

## 12. 速查表（一张图记住全章）

```
┌─────────────────────────────────────────────────────────────┐
│                直连路径 vs Sanctioned 路径                    │
├─────────────────────────────────────────────────────────────┤
│  Direct Route                                                │
│    IR ───────────────────► daemon compile+sign ──► kernel    │
│    (无 entitlement, 无 segmenter, 无 container)              │
│                                                              │
│  Sanctioned Route (Core ML)                                  │
│    Model ─► segmenter ─► container ─► loader-tier ─► daemon  │
│                                                              │
│  两条路径 → 同一块引擎；entitled 不多引擎，只多 loader         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              四项门控特性 + 失败层                            │
├──────────────────────┬──────────────────────────────────────┤
│ 3D conv              │ backend lowering 未实现                │
│ Native stateful      │ 硬件 primitive 缺失（最强）            │
│ bf16 program I/O     │ serialization dtype enum               │
│ Symbolic shapes      │ runtime path（compiler gate 是 on 的） │
├──────────────────────┴──────────────────────────────────────┤
│  共同模式：上一层接受，下一层拒绝                            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              最底层根：加载时签名                            │
│  kernel: corecrypto(prog bytes) + trustcache(vnode)          │
│  失败 → 0xe00002e2                                           │
│  唯一能加载的 = daemon compile+sign 的程序                   │
└─────────────────────────────────────────────────────────────┘
```

---

*本文档基于 arXiv:2606.22283v1 (21 Jun 2026) 第8章 (pp. 46–50) 撰写。论文为逆向工程参考文档，direct route 未被 Apple 官方支持、跨版本脆弱，仅适用于研究、测量和设备端实验，不应用于发布软件。*
