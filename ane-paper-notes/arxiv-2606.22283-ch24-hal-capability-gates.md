# arXiv:2606.22283 第24章《HAL and capability gates》深度解读

> 论文：*Apple Neural Engine: Architecture, Programming, and Performance*  
> 作者：Spencer H. Bryngelson  
> 单位：Georgia Institute of Technology  
> 提交：2026-06-21，302 页，12 图  
> 许可：CC-BY 4.0  
> DOI: `10.48550/arXiv.2606.22283`

---

## 0. 章节定位

第24章位于论文 **Part IV "Reverse-engineered internals"** 的第二章，承接第23章 (Program and container format)，下接第25章 (Compression internals)。

它回答一个工程问题：

> **一个编译器二进制如何为 A11–A18、M1–M5 共 28 个 target 生成代码？答案是一个 per-chip 数据表 + 两种 gate 机制。**

---

## 1. 全章主线 (SUMMARY)

> **一个编译器二进制 + 一张 per-chip 数据表 = 全 chip family 的代码生成。**

关键事实：

- 编译器是 **单一二进制**，任何 chip 都由同一份代码构建
- target 差异完全来自一张编译期读取的数据表：**HAL (Hardware Abstraction Layer)**
- HAL 包含两类条目：
  - **scalar 字段**（按 byte offset 索引的数值上限）
  - **capability byte**（密集布尔标志区，offset `0x48f`–`0x8cc`，每个 byte 按 `hal[offset] & 1` 读出，gate 一个 op 或一种 format）
- 操作合法性声明在 op 自身的 `MinimumFamily` trait 上：target family index ≥ N 才 native，低于 floor 走 decomposition
- **没有任何 compute op 的 floor 高于 A15** —— 新世代只加 core 数和时钟，不加新 op
- **Attested ≠ reachable**：表里的 capability 只证明"读它的那一层承认支持"，不证明"silicon 上能跑"。3D 卷积在 `0x70` attested kernel depth 16，但所有 device mask 上 lowering 都失败

---

## 2. HAL 表的结构 (§24.1)

HAL（struct 名为 `ZinIrHalParameters`）是一个 **packed 结构**，编译器对每个 target 调用对应的 constructor 构建。两种条目共存：

### 2.1 Scalar region（数值限制）
- offset 范围：`0x18` ~ 约 `0x348`（plain data）
- 非 scalar 成员（format map、cost-model curve）延伸到更后

### 2.2 Capability-byte region（单字节开关）
- offset 范围：`0x48f` ~ `0x8cc`
- 共约 **165 个单字节 flag**
- 其中 **24 个已恢复字段名**，其余按 offset 枚举 + 按 enabling family 分类
- 读取约定：`hal[offset] & 1`

### 2.3 Cost-model 区（嵌在同一表里）
- policy-name 字符串 @ `0x580`
- frequency-to-efficiency 曲线 @ `0x7a8`
- per-cycle divisor @ `0x228`
- 第18章 roofline 直接从这里读，**而不是单独文件**

### 2.4 代表性 scalar 字段表 (Table 24.1，节选)

| Offset | 字段 | M1 (H13) 值 | 何时变化 |
|---|---|---|---|
| `0x70` | `max_large_conv_kernel_dim_z` (3D conv kernel depth) | 16 | A13 起 attested（其下一档为 1） |
| `0x138` | `max_tensor_width` | 16384 | A16 起 65536 |
| `0x158` | `max_tensor_depth` | 16384 | A13 下一档 1，A16 起 65536 |
| `0x1b8` | `max_operand_bytes` (SRAM working set) | **2 MB** | 全线恒定（M9 为 1 MB） |
| `0x1c0` | `dram_alignment` (DMA granule) | 16 | 全线恒定 |
| `0x1c8` | `l2_bank_align` | 64 | 恒定 |
| `0x1f0` | L2-resident buffer threshold | 0 | A15: 32768，A16: 262144 |
| `0x200` | **dense kernel-memory cap** | **64 KB** | fold-path 预算 |
| `0x210` | **streamed kernel-memory cap** | **16 MB** | stream-path 预算 |
| `0x218` | instruction alignment | 256 | A14 起 16 |
| `0x228` | `ne_perf_cycle_divisor` | 64 | H11: 32，M9: 16 |
| `0x238` | `num_nes` (NE 核数) | 4 | die-keyed: 4/8/16/32/64 |
| `0x288` | extended dual-kernel-memory mode | 0 | 全 28 target 都为 0 |
| `0x3f0` | reduction-via-transpose extent | 192 | A15 起 384 |
| `0x400` | `pe_min_patch_width_log2` | 4 (即 16 像素) | M1/M5 恒定 |
| `0x580` | cost-model policy name | "Simple"/"None" | 旧/小 profile 不同 |
| `0x668` | interchange-format count | 3 | A14: 13，A15: 16，A16: 14 |

### Listing 24.1 C 视图（关键片段）

```c
struct ZinIrHalParameters {
    uint64_t max_large_conv_kernel_dim_z;  /* 0x70: 3D-conv kernel depth = 16 */
    uint64_t max_tensor_width;             /* 0x138: 16384 */
    uint64_t max_operand_bytes;            /* 0x1b8: 2 MB */
    uint64_t dram_alignment;               /* 0x1c0: 16 */
    uint64_t l2_bank_align;                /* 0x1c8: 64 */
    uint64_t num_nes;                      /* 0x238: 4 */
    uint8_t  cap_bytes[0x8cd - 0x48f];     /* 0x48f..0x8cc: per-op cap flags */
};
```

读取习惯：`texture engine @ 0x81d`、`kernel-streaming master @ 0x48f` 都是 `hal[offset] & 1`。

---

## 3. Operation gate：MinimumFamily trait (§24.2)

**操作合法性不在 HAL 表里，而在 op 自身上** —— 编译器给每个 backend op 附着一个 trait：`MinimumFamily`。

### 3.1 Family index 编号

| Generation | Family index |
|---|---|
| A11 Legacy | 0 |
| A12 | 1 |
| A13（含 M1） | 2 |
| A14 | 3 |
| A15 | 4 |
| … | … |
| A17（含 M5） | 6 |
| A18 | 7 |

### 3.2 Native-or-decompose 决策 (Listing 24.2)

```python
# mlir::OpTrait::anec::MinimumFamily: native iff target family >= N
def op_is_native(op, target_family):
    return target_family >= op.minimum_family   # 例 softmax N=2(A13), sin N=4(A15)

def lower_op(op, target_family):
    if op_is_native(op, target_family):
        emit_native(op)        # 一条 anec op
    else:
        decompose(op)          # 重写成 floor 以下合法的 op
```

**M1 = family 2，M5 = family 6**：所以 `sin` (floor 4) 在 M5 上 native、在 M1 上 decompose。

### 3.3 Floor 分档 (Table 24.2)

| Floor | Native on | 代表性 op |
|---|---|---|
| **F0** | 所有 family | conv、matmul、pooling、elementwise、reshape、transpose、concat |
| **F2** (A13+) | A13 起 | softmax、layer/instance/batch norm、reductions、attention、erf、sqrt |
| **F3** (A14+) | A14 起 | crop-resize、resample |
| **F4** (A15+) | A15 起 | sin、cos、global argmin/argmax |

> **关键观察**：没有 compute op 的 floor 高于 A15。**新世代只加 core 数和时钟，不加新 op。**

### 3.4 两种 gate 的分工

| Gate | 粒度 | 决定什么 |
|---|---|---|
| **Capability byte** (`hal[offset] & 1`) | op **内部**某条路径 | 例：texture engine @ `0x81d` 是否存在；M1 上读 0 → resize 走 decomposition |
| **MinimumFamily trait** | op **本身是否 native** | target family 是否 ≥ floor |

任何一个 gate 关闭时，编译器要么：
- emit 一个 decomposition 成合法 op
- 要么 reject，报错信息会点名架构名

### 3.5 Per-chip 差异是数据，不是代码

> 因为 floor 是 op 的属性、limit 是 chip 的表项，**per-chip 差异是 data，不是 code**。重写 op 的编译器文本在全 family 上 **完全相同**，chip 只是在底下的表里挑不同的 limit/gate/decomposition 策略。

这是 Apple 把 28 个 target 收敛到一个二进制的根本设计原则。

---

## 4. Capability-byte 全线视图 (§24.3, Table 24.3)

| Byte | Gate 含义 | M1 (H13) | A14 | A15 | A16 | A18 |
|---|---|---|---|---|---|---|
| `0x48f` | **kernel-streaming master**（64KB↔16MB 选择） | 1 | 1 | 1 | 1 | 1 |
| `0x494` | square-after-reduction fusion | 0 | 1 | 1 | 1 | 1 |
| `0x4a9` | dropout and random | 0 | 0 | 1 | 1 | 1 |
| `0x4f2` | global argmin/argmax | 1 | 1 | 1 | 1 | 1 |
| `0x529` | per-format kernel-stride enable（palette stream） | 1 | 1 | 1 | 1 | 1 |
| `0x52d` | **fp8 E4M3 kernel format** | 0 | 0 | 0 | 0 | **1** |
| `0x563` | **FIFO-mode DMA** | 0 | 0 | 0 | 0 | **1** |
| `0x815` | softmax, native | 1 | 1 | 1 | 1 | 1 |
| `0x816` | instance normalization, native | 1 | 1 | 1 | 1 | 1 |
| `0x81a` | local-response normalization, native | 1 | 1 | 1 | 1 | 1 |
| `0x81d` | **texture engine** | **0** | 1 | 1 | 1 | 1 |

### 三个值得记住的 byte

1. **`0x81d` texture engine**：M1 上最大的功能缺口。读 0 → resize/crop-resize/resample/affine transform/hardware gather/symmetric padding **全部走软件 decomposition**。从 A14 起置 1
2. **`0x52d` fp8 E4M3**：28 个 target 里 **只有 A18** 设；M5（A17）**没有**
3. **`0x48f` streaming master** 和 **`0x529` palette stream**：M1 上都为 1，所以 int4 palette 和 sparse form 流式可用；int8 和 blockwise 走 fold 路径（见第25章）

### Host 可以恢复任何 chip 的表

> 编译器通过调用 target 的 constructor 构建表，**单一 host 即可恢复整条 chip line 的表，无论它运行的是不是这块芯片**。

这是逆向工程能够批量产 per-chip target 表的根本原因。

---

## 5. Per-family scalar matrix (§24.4, Table 24.4)

跨世代 anchor 的 scalar 参数呈现同一模式：**一个值持稳几代，然后一次性跳变**。

| 字段 (offset) | M1 (H13) | A14 | A15 | A16 (M4) | A17 (M5) |
|---|---|---|---|---|---|
| `num_nes` (`0x238`) | 4 | 4 | 4 | 4 | **16** |
| `max_operand_bytes` (`0x1b8`) | 2 MB | 2 MB | 2 MB | 2 MB | 2 MB |
| `max_tensor_width` (`0x138`) | 16384 | 16384 | 16384 | **65536** | 65536 |
| `max_tensor_depth` (`0x158`) | 16384 | 16384 | 16384 | **65536** | 65536 |
| `max_large_conv_kernel_dim_z` (`0x70`) | 16 | 16 | 16 | 16 | 16 |
| L2-resident threshold (`0x1f0`) | 0 | 0 | **32768** | **262144** | 262144 |
| instruction alignment (`0x218`) | 256 | **16** | 16 | 16 | 16 |
| reduction-transpose extent (`0x3f0`) | 192 | 192 | **384** | 384 | 384 |
| interchange-format count (`0x668`) | 3 | **13** | **16** | **14** | 14 |

> **M5 列 `num_nes = 16`** 是因为这是 16-core Pro-class profile；base A17 是 4 核。per-die 序列：4 (base) / 8 (g-suffix) / 16 (s 和 legacy 16-core) / 32 (c) / 64 (d Ultra-class)。

---

## 6. Kernel-memory split (§24.5, Listing 24.3)

streaming master byte `0x48f` 不只 gate 压缩权重流——它 **选择两层 kernel-memory cap 中的哪一层** 来衡量 layer 的权重：

```python
# ExceedKmemSizeLimit: split-legalize a layer's weights when they exceed the cap
def exceeds_kmem(hal, demand, is_streamable):
    cap = hal[0x210] if (is_streamable and hal[0x48f]) else hal[0x200]
    return cap < demand
    # 0x200 = 64 KB dense,  0x210 = 16 MB streamed
```

### 后果

| 情形 | M1 上的行为 |
|---|---|
| 普通 dense 权重 > 64 KB | **split 成多个 sub-layer**，dispatch 数和编译时间上升 |
| 任意权重 > 16 MB | 同样 split |
| streamable 压缩权重（且 master byte = 1） | 用 16 MB cap，每层权重容量大幅提升 |

> **注意**：这是 weight path 的限制。**activation 不受此限**，它们仍受 `max_tensor_*` 维度 cap 约束；小权重 + 大 activation 的 layer 受 partition pass 的 tiling 成本约束，而不是这个离散限制。

---

## 7. Dead and family-gated fields (§24.6)

**不是表里每个值都是 live gate。** 对 28 个 target blob 做 byte-granular re-diff 后：
- **0 个未解码 scalar 字段**
- 但若干 offset 虽然 per-family 变化、却 **从不通过表指针被读回** —— 它们是 **write-only mirror**

### 五个 dead scalar offset

| Offset | 内容 |
|---|---|
| `0x18` | global element cap |
| `0x80` | kernel-depth constant |
| `0x260` | legacy tiling granule |
| `0x320` | (未命名) |
| `0x29c` | die-class flag |

每个都随 family 变化，但 reader 实际从 **另一个共享 byte displacement 的对象** 读：tensor-dimensions、compiler-parameters、memory-pools 结构。

### Distinguishing test

> 表字段读取的判据：**access 处的 base register 是否持有 table pointer**。因为同一 displacement 会别名 dozens 个其他 by-reference 结构——单看 displacement 匹配 **不能** 证明是表读取。

### 反例：`0x563` FIFO-mode

读 0（M1），但 **不是 dead**：它在 A18 上置 1，通过表指针读取，但读取处在一个 **只有当 byte 已置位才进入** 的分支下。

> **教训**：per-family 值模式本身不能建立 live gate——只有 **traced reader off the table base** 才能证明。

---

## 8. 命名其余 capability flag (§24.7)

### 8.1 为什么没有 per-field getter

`ZinIrHalParameters` **没有 per-field getter 方法**。编译器读取方式是 **inline 的 `ldrb` / `ldr [reg, #offset]`**。所以第一轮逆向只能恢复 offset 和 value，**不能恢复 name**。

### 8.2 名字唯一存活的地方

编译器保留了 **full mangled C++ symbols**。Reader 函数签名形如 `ZinIrHalParameters const&`，所以 **reader 函数的名字 = 它读取字段的 label**。

判据：只有当 load 的 base register 是函数的 `ZinIrHalParameters const&` 参数时，才把这次读归因于表。

### 8.3 本轮新命名的 flag (Table 24.5 节选)

| Offset | 字段 | Reader 函数 |
|---|---|---|
| `0x4a8` | PE work-unit-shape supported | `PERasterization::ComputeWUShape` |
| `0x4ac` | small-source-mode compression | `ZinANELayer::AllowCompressionBasedOnSmallSourceMode` |
| `0x4b0` | non-power-of-2 WU width | `NERasterization::CanUseNonPowerOf2WUs` |
| `0x4f0` | preferred kernel layout format | `ZinIrKernel::GetPreferredKernelLayoutFormat` |
| `0x500` | transpose and multicast config | `ZinNELayer::FindValidMirInfoForTransposeCore` |
| `0x520` | secure-mode cache-hint DSID gate | `GetDSIDFromPriorityHalAndSecureMode` |
| `0x52c` | tensor-format flag (pairs with `0x52d` fp8) | `ZinLayerValidationUtils::ValidateFormat` |
| `0x54c` | cache-prefetch kernel-task interval limit | `ZinValidateTd<17>::ValidateCachePrefetchKernelTaskInterval` |
| `0x5a8` | cache-hint DSID value | `GetDSIDFromPriorityHalAndSecureMode` |
| `0x708` | reflective-padding max extent | `ZinValidateTd<20>::ValidateReflectivePaddingMode` |
| `0x748` | gather/texture-engine descriptor ptr | `ZinGatherLayer::CreateTELayer` |
| `0x8b4` | tile-height-errata threshold | `ZinTileHeightErrata::Workaround` |
| `0x8bc` | chaining enabled | `ZinIrRegAllocUtil::IsChainable` |
| `0x8e0` | kernel-caching enabled | `ZinIrTdValidationUtil::ValidateKernelCaching` |

### 8.4 三种命名精度

| 精度 | 数量 | 含义 |
|---|---|---|
| **精确含义** | ~30 | base register 已对表参数校验 |
| **class-named** | 其余 ~65 | reader 标识 subsystem，但具体语义未明（例：`ZinValidateTd::CheckInRangeDmaAccess` 的 per-axis DMA 范围 bound） |
| **未命名** | — | 在 `0x820`–`0x8f8` 区间被 `0x81d` texture-engine byte gate 的 plane-equation 系数等 |

### 8.5 两个被拒候选

| Offset | 拒绝原因 |
|---|---|
| `0xcf8` | `adrp` 形成的 read-only constant 读取，**不** off table |
| `0x678` | 嵌套对象，**两指针深**，不 off table |

### 8.6 Struct 全貌

> 整个 struct 大小 **`0x938` bytes**。其中非 capability 的少数条目是 **cost-model 系数块** `0x580`–`0x7f0`（freq→efficiency 曲线、rate index、perf multiplier，第18章 roofline 直接读）。
>
> **`0x938` 之后没有任何表**：早期阅读曾在 `0xa30`–`0xe84` 定位一个 A12 op-emulation catalog，那是 **struct 之外相邻 zeroed memory 的误读**，不是真实字段。

---

## 9. 全章灵魂：Attested is not reachable (§24.8)

这是全章最重要、也是贯穿全文 (§4.4, §8.2) 的判据。

> **HAL 表里 attested 的 capability 只证明"读它的那一层承认支持"，不证明"op 能 lower 成 task descriptor 并在 silicon 上跑"。这是两个层。第一层 attested 的 capability 可以在第二层失败。**

### 固定规则的反例：3D 卷积

| 层 | 状态 |
|---|---|
| HAL scalar `0x70` | attested 3D-conv kernel depth = 16 |
| 编译器 frontend | 识别该操作 |
| backend lowering | **每个 device mask 都失败**，返回 "not supported on any backend" |

→ capability 在表里，op 不运行。

### 反向 gap：checker 通过但 codegen 拒绝

M1 上 `top-k`、`sort`、`dynamic-slice` 的 validator 都可调用，**但三个都在 code generation 被拒**。

### 综合判据

> 表里的 bit、识别 op 的 frontend、pass 的 validator —— 每一个都只是 **关于某一层的 claim**。只有 **target 上的 compile-and-run** 才能确认 op 在执行层真的可达。

这就是为什么：
- 第4章的 reachable surface **小于** 表 advertise 的 surface
- 第4章每个 native 条目都是在 M1 上 **编译并运行过** 验证的，不是从 capability byte 推断的

---

## 10. 三个值得记住的判断

1. **"Per-chip 差异是数据，不是代码"**：Apple 把 28 个 target 收敛到一个二进制 + 一张表，这是 ANE 工具链最优雅的设计决策。任何"为不同 chip 编译"本质上是"换表 + 走同一份 op 重写代码"。

2. **两种 gate 是分工的，不是冗余的**：
   - `MinimumFamily` 决定 **op 是否 native**
   - `capability byte` 决定 **op 内某条路径是否启用**
   - 任一关闭 → decompose 或 reject。它们叠加，不互相替代

3. **"Attested is not reachable" 是逆向工程的根本方法论警示**：静态分析（读表、读 frontend、读 validator）只能给"可能可达"的下界，**只有动态测量能给上界**。这是为什么作者反复强调 measured vs decompile-derived vs predicted 的区分。

---

## 11. 与其他章节的勾连

- **→ 第4章 (Capability surface)**：§4.4 "Attested is not reachable" 在 §24.8 得到完整机制解释；第4章的每个 native 条目都是 compile-and-run 验证过的
- **→ 第8章 (Entitlement boundary)**：§8.2 的 counter-and-event engine stub 是 "attested is not reachable" 的最强形式——硬件 primitive 缺失
- **→ 第18章 (Optimization and cost model)**：cost-model 系数块 `0x580`–`0x7f0` 嵌在 HAL 内部，roofline 直接读
- **→ 第25章 (Compression internals)**：streaming master `0x48f`、palette stream `0x529` 决定 int4/int8/blockwise/sparse 走 stream 还是 fold 路径
- **→ 第34章 (Cross-silicon targets)**：HAL 是 28 个 target 的统一来源；第34章的 per-family table 都源自这张表
- **→ 第35章 (Per-family code generation)**：§35.1 "One binary, eight families" 的根基就是本章的 HAL 设计；§35.6 task-descriptor 的 per-generation field offset 也是 HAL 字段
- **→ 第20章 (Datapath and MAC geometry)**：geometry constant（如 `pe_min_patch_width_log2 @ 0x400`）来自 HAL

---

## 12. 速查图（一张图记住全章）

```
┌──────────────────────────────────────────────────────────────┐
│   单一编译器二进制 → 全 chip family 代码生成                    │
│                                                               │
│        target = HAL 表（编译期 per-chip 构建）                 │
│   ┌───────────────────────────────────────────────┐           │
│   │ ZinIrHalParameters  (struct, 0x938 bytes)     │           │
│   │                                                │           │
│   │  scalar region  0x18 ~ 0x348                  │           │
│   │    - max_tensor_width, num_nes, ...           │           │
│   │    - dense cap 0x200 (64KB)                   │           │
│   │    - streamed cap 0x210 (16MB)                │           │
│   │                                                │           │
│   │  cost-model block  0x580 ~ 0x7f0              │           │
│   │    (freq→eff curve, divisor, roofline)        │           │
│   │                                                │           │
│   │  capability bytes  0x48f ~ 0x8cc (165 个)     │           │
│   │    hal[offset] & 1                            │           │
│   │    0x48f streaming master                     │           │
│   │    0x52d fp8 (仅 A18)                         │           │
│   │    0x81d texture engine (M1=0, A14+=1)        │           │
│   └───────────────────────────────────────────────┘           │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│   两种 gate（分工，不冗余）                                    │
│                                                               │
│   MinimumFamily trait  →  op 是否 native                      │
│       F0: 全 family (conv/matmul/...)                         │
│       F2: A13+ (softmax, norm, attention)                     │
│       F3: A14+ (crop-resize, resample)                        │
│       F4: A15+ (sin, cos, argmin/argmax)                      │
│       没有任何 compute op floor > A15                          │
│                                                               │
│   capability byte      →  op 内部某条路径                      │
│       hal[offset] & 1                                          │
│                                                               │
│   任一关闭 → decompose 或 reject                              │
│   per-chip 差异 = 数据，不是代码                               │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│   灵魂判据：Attested ≠ Reachable                              │
│                                                              │
│   层 1: HAL 表 attested    ───┐                              │
│   层 2: frontend 识别        ─┤ 每个都是"某一层的 claim"      │
│   层 3: validator 通过      ──┘                              │
│   层 4: backend lowering    ───  才是真相                     │
│                                                              │
│   反例 1: 3D conv @ 0x70=16  →  lowering 全失败               │
│   反例 2: top-k/sort/dynamic-slice validator 通过但 codegen 拒 │
│                                                              │
│   ⇒ 只有 compile-and-run 在 target 上才能确认                 │
│   ⇒ 第4章 reachable surface < HAL advertise 的 surface        │
└──────────────────────────────────────────────────────────────┘
```

---

*本文档基于 arXiv:2606.22283v1 (21 Jun 2026) 第24章 (pp. 150–157) 撰写。论文为逆向工程参考文档，direct route 未被 Apple 官方支持、跨版本脆弱，仅适用于研究、测量和设备端实验，不应用于发布软件。*
