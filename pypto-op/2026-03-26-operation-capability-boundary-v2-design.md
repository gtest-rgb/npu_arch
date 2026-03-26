# Operation 模块能力边界判断体系设计文档 V2

> 版本：2.1
> 日期：2026-03-26
> 作者：Operation 模块负责人
> 前置文档：2026-03-26-operation-capability-boundary-system-design.md（V1，已废弃）

## 1. 背景与目标

### 1.1 问题背景

Operation 模块是 PyPTO 框架的核心组件，提供 90+ 个操作 API、200+ OP_CODE。作为模块负责人，需要快速回答以下问题：

- "matmul 支持 bf16 吗？K 维度有什么约束？"
- "OP_ADD 对输入的 stride 有什么要求？数据必须连续吗？"
- "这个网络（如 Attention）在 PyPTO 中是否支持？"

**当前痛点：**
- 约束条件分散在 opcode.h、opcode.cpp、tileop/ 源码中，没有统一记录
- 一个 Operation 可能展开为多个 OP_CODE，每个 OP_CODE 的 TileOp 实现有不同的数据排布约束
- 回答问题依赖经验 + 查代码，效率低且易遗漏

### 1.2 目标

建立一套**三层约束知识库**，实现：

1. **快速查询**：输入 OP 名或 OP_CODE，输出结构化约束报告
2. **多维覆盖**：覆盖 Layout、Stride 对齐、地址对齐、硬件参数、TileShape 五个约束维度
3. **完整 pipeline**：覆盖 Operation 的完整执行链路（CopyIn → 片上搬运 → 计算 → CopyOut），而非仅计算核心
3. **双用户支持**：人类可直接阅读 YAML 和 INDEX.md；AI Skill 可自动查询生成报告
4. **可追溯**：每条约束标注来源（代码文件 + 经验）

### 1.3 范围

**在范围内：**
- 计算类 OP：Vector Elementwise/Binary/Reduce、Cube Matmul/Conv、Sort
- 数据搬运类 OP：CopyIn/Out、MTE、Load、Gather/Scatter
- View/Assemble 类 OP：Reshape、View、Assemble、Expand
- 数据转换类 OP：Cast、Convert、Transpose

**不在范围内：**
- 同步类 OP：OP_SYNC_SRC/DST、OP_BAR_V/M、OP_PHASE1/2
- 通信类 OP：SHMEM_*、MOE_*、FFN_*、SEND_TO_*、DISPATCH_*
- 内存分配类 OP：*_ALLOC
- 网络场景索引（不在本设计中，可后续扩展）

### 1.4 与 V1 的区别

| 维度 | V1 | V2（本文档） |
|------|-----|-----|
| 分层结构 | 四层（含网络场景） | 三层（Operation API → OP_CODE → TileOp） |
| 约束存储 | YAML + Markdown 混合 | 纯 YAML，按 OP 类别分目录 |
| Operation 文档 | 每个 OP 一个 Markdown 文件 | 去掉，由 opcode_mapping.yaml + TileOp YAML 覆盖 |
| 目录组织 | 按功能平铺 | 按类别（vector/cube/move/view/misc）分目录 |
| 约束维度 | 排布 + stride + 对齐 | Layout + Stride 对齐 + 地址对齐 + 硬件参数 |

---

## 2. 整体架构

```
┌──────────────────────────────────────────────────────────────┐
│               AI Skill: pypto-op-capability-checker           │
│  输入: OP名/OP_CODE/能力查询                                  │
│  输出: 结构化约束报告（带依据，覆盖完整 pipeline）             │
└──────────────────────────────────────────────────────────────┘
                           ↓ 查询
┌──────────────────────────────────────────────────────────────┐
│                    约束知识库（三层结构）                       │
│                                                               │
│  Layer 1: Operation API + Pipeline                           │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ opcode_mapping.yaml                                   │    │
│  │                                                       │    │
│  │ matmul:                                               │    │
│  │   opcodes: [OP_A_MUL_B, ...]                          │    │
│  │   pipeline:                                            │    │
│  │     CopyIn(GM→L1) → Load(L1→L0A/B) → MMAD →         │    │
│  │     Extract(L0C→L1) → CopyOut(L1→GM)                 │    │
│  │   每个 stage 引用 constraint_file                     │    │
│  └──────────────┬───────────────────────────────────────┘    │
│                 ↓                                              │
│  Layer 2: OP_CODE → TileOp 约束                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ tileop_constraints/                                   │    │
│  │  ├── _common.yaml        # 公共硬件常量                │    │
│  │  ├── vector/             # Elementwise/Binary/Reduce   │    │
│  │  ├── cube/               # Matmul/Conv                 │    │
│  │  ├── move/               # CopyIn/Out, L1↔L0 搬运     │    │
│  │  ├── view/               # Reshape/View/Assemble       │    │
│  │  └── misc/               # 其他 OP                     │    │
│  │                                                       │    │
│  │  每个 OP_CODE: Layout + Stride + Alignment + Hardware │    │
│  │              + TileShape（约束从 hardware 推导）       │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                               │
│  Layer 3: 人类入口                                             │
│  ┌──────────────────────┐                                    │
│  │ INDEX.md              │  总索引 + 快速查询指南             │
│  └──────────────────────┘                                    │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. 目录结构

```
.agents/skills/pypto-op/
├── pypto-op-capability-checker/        # AI Skill
│   ├── SKILL.md                        # Skill 定义
│   └── references/
│       ├── query_guide.md              # 查询模式说明
│       └── report_template.md          # 输出报告模板
│
└── constraints/                         # 约束知识库
    ├── INDEX.md                         # 总索引（人类入口）
    ├── opcode_mapping.yaml              # Layer 1: Operation → OP_CODE
    │
    └── tileop_constraints/              # Layer 2: TileOp 约束
        ├── _common.yaml                 # 公共硬件常量
        ├── vector/
        │   ├── elementwise.yaml         # ADD, SUB, MUL, DIV, EXP, NEG, RELU, SQRT, ABS, LN...
        │   ├── elementwise_scalar.yaml  # ADDS, SUBS, MULS, DIVS, MAXS, MINS...
        │   ├── broadcast.yaml           # ADD_BRC, SUB_BRC, MUL_BRC, DIV_BRC, MAX_BRC, MIN_BRC
        │   ├── reduce.yaml              # ROWMAX, ROWSUM, ROWEXPMAX, ROWEXPSUM, ROWSUMLINE...
        │   ├── cast.yaml                # CAST, CONVERT
        │   ├── gather_scatter.yaml      # GATHER, SCATTER, GATHER_ELEMENT, SCATTER_ELEMENT...
        │   ├── compare.yaml             # CMP, CMPS, MAXIMUM, MINIMUM, PAIRMAX, PAIRMIN...
        │   ├── logical.yaml             # LOGICALNOT, LOGICALAND, WHERE_TT, WHERE_TS...
        │   ├── bitwise.yaml             # BITWISEAND, BITWISEOR, BITWISERIGHTSHIFT...
        │   └── sort.yaml                # TOPK, ARGSORT, MRGSORT, BITSORT...
        ├── cube/
        │   ├── matmul.yaml              # A_MUL_B, A_MULACC_B, A_MUL_BT, A_MULACC_BT, AT_MUL_B, AT_MUL_BT
        │   └── conv.yaml                # CONV, CONV_ADD, CUBE_CONV_D2S...
        ├── move/
        │   ├── copy_in.yaml             # COPY_IN, UB_COPY_IN, L1_COPY_IN, TRANSPOSE_MOVEIN...
        │   ├── copy_out.yaml            # COPY_OUT, UB_COPY_OUT, L0C_COPY_OUT, TRANSPOSE_MOVEOUT...
        │   ├── local_move.yaml          # L1_COPY_UB, UB_COPY_L1, COPY_UB_TO_UB, L1_COPY_UB...
        │   └── gather_load.yaml         # GATHER_IN_L1, GATHER_IN_UB, LOAD, VLD, VST...
        ├── view/
        │   ├── reshape.yaml             # RESHAPE, RESHAPE_COPY_IN, RESHAPE_COPY_OUT
        │   ├── view.yaml                # VIEW, VIEW_TYPE
        │   ├── assemble.yaml            # ASSEMBLE, ASSEMBLE_SSA
        │   └── expand.yaml              # EXPAND, EXPANDEXPDIF
        └── misc/
            ├── duplicate.yaml           # DUPLICATE, VEC_DUP
            ├── register.yaml            # REGISTER_COPY, VLD, VST
            ├── index.yaml               # INDEX_OUTCAST, INDEX_PUT, INDEX_ADD
            └── special.yaml             # CONCAT, CUM_SUM, ONEHOT, PAD, COMPACT, RANGE, HYPOT, PRELU, POW...
```

---

## 4. YAML 数据格式

### 4.1 `_common.yaml` — 公共硬件常量

```yaml
hardware_constants:
  BLOCK_SIZE: 32          # 字节，一个 block 的最小单位
  BLOCK_NELEM_B16: 16     # bfloat16/half，一个 block 的元素数
  BLOCK_NELEM_B32: 8      # float32，一个 block 的元素数
  MASK_LEN: 64            # vector mask 长度
  REPEAT_MAX: 255         # 单次 repeat 最大次数
  REPEAT_STRIDE_MAX: 255  # repeat stride 最大值
  DUP_REPEAT_STRIDE_MAX: 4095  # dup repeat stride 最大值
  BLOCK_NUM_ONE_REPEAT: 8 # 一次 repeat 最多处理的 block 数
  NBLOCK_PER_MASK_B16: 4  # bf16 一个 mask 覆盖的 block 数
  REPEAT_BYTE: 256        # 一次 repeat 处理的最大字节数

alignment_rules:
  UB: 32                  # Unified Buffer，32B 对齐
  L1: 32                  # L1 Buffer，32B 对齐
  L0A: 32                 # L0A Buffer，32B 对齐
  L0B: 32                 # L0B Buffer，32B 对齐
  L0C: 32                 # L0C Buffer，32B 对齐
  GM: 32                  # Global Memory，32B 对齐

dtype_constraints:
  fp16:
    element_size: 2
    block_nelem: 16
  bf16:
    element_size: 2
    block_nelem: 16
  fp32:
    element_size: 4
    block_nelem: 8
  int8:
    element_size: 1
    block_nelem: 32
```

### 4.2 `opcode_mapping.yaml` — Operation → OP_CODE 映射

一个高层 Operation（如 `matmul`）在实际执行时会展开为一个 **OP_CODE pipeline**，包含数据搬运（CopyIn/Out）、片上 buffer 搬运（L1→L0）、计算核心（MMAD）等环节。`opcodes` 字段记录该 Operation 可能用到的所有计算 OP_CODE，`pipeline` 字段记录完整的执行链路。

```yaml
matmul:
  description: "矩阵乘法"
  category: cube
  opcodes:
    - opcode: OP_A_MUL_B
      condition: "默认情况，C = A * B"
      core_type: AIC
    - opcode: OP_A_MULACC_B
      condition: "累加场景，C += A * B"
      core_type: AIC
    - opcode: OP_A_MUL_BT
      condition: "B转置，C = A * B^T"
      core_type: AIC
    - opcode: OP_A_MULACC_BT
      condition: "B转置累加，C += A * B^T"
      core_type: AIC
  # 完整执行 pipeline，按执行顺序列出所有环节
  pipeline:
    - stage: copy_in_a
      opcode: OP_COPY_IN
      description: "输入 A: GM → L1"
      direction: "GM → L1"
      constraint_file: "move/copy_in.yaml"
    - stage: copy_in_b
      opcode: OP_COPY_IN
      description: "输入 B: GM → L1"
      direction: "GM → L1"
      constraint_file: "move/copy_in.yaml"
    - stage: load_a
      opcode: OP_L1_TO_L0A
      description: "输入 A: L1 → L0A"
      direction: "L1 → L0A"
      constraint_file: "move/copy_in.yaml"
    - stage: load_b
      opcode: OP_L1_TO_L0B
      description: "输入 B: L1 → L0B"
      direction: "L1 → L0B"
      constraint_file: "move/copy_in.yaml"
    - stage: mmad
      opcode: OP_A_MUL_B
      description: "矩阵乘法 C = A * B"
      direction: "L0A,L0B → L0C"
      constraint_file: "cube/matmul.yaml"
    - stage: extract_c
      opcode: OP_L0C_COPY_OUT
      description: "输出 C: L0C → L1"
      direction: "L0C → L1"
      constraint_file: "move/copy_out.yaml"
    - stage: copy_out
      opcode: OP_COPY_OUT
      description: "输出 C: L1 → GM"
      direction: "L1 → GM"
      constraint_file: "move/copy_out.yaml"
  source: "opcode.h:L158-163, operation_impl.cpp"

add:
  description: "逐元素加法"
  category: vector
  opcodes:
    - opcode: OP_ADD
      condition: "默认，两个 tensor 相加"
      core_type: AIV
    - opcode: OP_ADD_BRC
      condition: "broadcast 标量加法"
      core_type: AIV
    - opcode: OP_ADDS
      condition: "立即数标量加法"
      core_type: AIV
  pipeline:
    - stage: copy_in
      opcode: OP_COPY_IN
      description: "输入: GM → UB"
      direction: "GM → UB"
      constraint_file: "move/copy_in.yaml"
    - stage: compute
      opcode: OP_ADD
      description: "逐元素加法"
      direction: "UB → UB"
      constraint_file: "vector/elementwise.yaml"
    - stage: copy_out
      opcode: OP_COPY_OUT
      description: "输出: UB → GM"
      direction: "UB → GM"
      constraint_file: "move/copy_out.yaml"
  source: "opcode.h:L96, L105, L58"
```

**字段说明：**
- `category`: 对应 tileop_constraints 下的子目录名
- `opcodes[].condition`: 什么情况下触发该 OP_CODE
- `opcodes[].core_type`: 执行核心（AIC = Cube, AIV = Vector）
- `pipeline[].stage`: 环节名称（copy_in, compute, copy_out 等）
- `pipeline[].opcode`: 该环节使用的 OP_CODE
- `pipeline[].direction`: 数据流向（如 GM → L1 → L0A）
- `pipeline[].constraint_file`: 该 OP_CODE 的约束文件路径，AI Skill 据此查询各环节约束
- `source`: 约束来源文件

**设计要点：**
- `pipeline` 是**逻辑视图**，实际代码生成可能对某些环节做融合或省略（如 UB 已有数据时跳过 CopyIn）
- `constraint_file` 指向 `tileop_constraints/` 下的文件，AI Skill 通过它查询每个环节的 Layout/Stride/Alignment 约束
- 一个 Operation 的整体能力边界 = pipeline 中**所有环节**约束的交集

### 4.3 TileOp 约束 YAML 格式

每个 `tileop_constraints/*.yaml` 文件内，每个 OP_CODE 一条记录：

```yaml
OP_ADD:
  description: "逐元素加法"
  calc_type: BROADCAST
  core_type: AIV
  tileop: "TileOp::Tadd"
  pipe: [PIPE_V, PIPE_V]

  # dtype 支持情况
  dtype:
    fp16: supported
    bf16: unsupported
    fp32: supported
    int8: supported

  # 每个输入/输出的约束
  operands:
    input_a:
      mem_type: [UB]
      layout:
        type: "任意"
        contiguous: false
        max_dims: 5
      stride:
        required: false
        constraints: []
      alignment:
        address: null
        scope: null
      shape:
        min_elements: 1
        max_elements_per_dim: null

    input_b:
      mem_type: [UB]
      layout:
        type: "任意"
        contiguous: false
        max_dims: 5
      stride:
        required: false
        constraints: []
      alignment:
        address: null
        scope: null

    output:
      mem_type: [UB]
      layout:
        type: "与输入一致"
        contiguous: false
        max_dims: 5
      stride:
        required: false
        constraints: []
      alignment:
        address: null
        scope: null

  # 硬件参数约束
  hardware:
    mask_len: 64
    repeat_max: 255
    block_size: 32
    elements_per_block:
      fp16: 16
      bf16: 16
      fp32: 8

  # 对应 opcode.h 中的集合标签
  tags: [BINARY_OPS, LASTUSE_OPS, SUPPORT_DYNAMIC_UNALIGNED_OPS]

  # 备注（经验、特殊情况）
  notes: |
    - 支持动态 unaligned shape
    - 支持 lastuse 优化
    - bf16 不支持（UNSUPPORT_BF16_OPS）

  # 依据来源
  source: "opcode.cpp RegisterVectorBinary, UNSUPPORT_BF16_OPS"
```

**有 stride 约束的示例（Cube 类）：**

```yaml
OP_A_MUL_B:
  description: "矩阵乘法 C = A * B"
  calc_type: MATMUL
  core_type: AIC
  tileop: "TileOp::Tmul"
  pipe: [PIPE_M, PIPE_M]

  dtype:
    fp16: supported
    bf16: supported
    fp32: supported

  operands:
    input_a:
      mem_type: [L0A]
      layout:
        type: "NZ格式"
        contiguous: false
        max_dims: 2
      stride:
        required: true
        constraints:
          - dim: 0
            constraint: "stride[0] 必须是 32B 的整数倍"
            reason: "MTE2 加载要求"
          - dim: 1
            constraint: "连续，不允许 stride"
            reason: "L0A 硬件限制，K 维必须连续存储"
      alignment:
        address: 32
        scope: "首地址"
      shape:
        min_m: 1
        min_k: 16

    input_b:
      mem_type: [L0B]
      layout:
        type: "NZ格式"
        contiguous: false
        max_dims: 2
      stride:
        required: true
        constraints:
          - dim: 0
            constraint: "连续，不允许 stride"
            reason: "L0B 硬件限制"
          - dim: 1
            constraint: "stride[1] 必须是 32B 的整数倍"
            reason: "MTE2 加载要求"
      alignment:
        address: 32
        scope: "首地址"

    output:
      mem_type: [L0C]
      layout:
        type: "NZ格式"
        contiguous: true
      stride:
        required: false
        constraints: []
      alignment:
        address: 32
        scope: "首地址"

  hardware:
    block_size: 32
    base_block_m: 16
    base_block_k: 16
    base_block_n: 16

  tags: [SUPPORT_DYNAMIC_UNALIGNED_OPS]

  notes: |
    - K 维度在 bf16 下必须 16 对齐
    - 输入必须通过 MTE 从 L1 加载到 L0A/L0B

  source: "cube/cube_pto.h, opcode.cpp RegisterCube"
```

### 4.4 Schema 字段参考

| 字段路径 | 类型 | 说明 |
|----------|------|------|
| `description` | string | OP_CODE 中文描述 |
| `calc_type` | enum | ELMWISE/BROADCAST/REDUCE/MATMUL/CONV/MOVE_IN/MOVE_OUT/MOVE_LOCAL/CAST/OTHER/SYS（SYNC/DISTRIBUTED 不在范围内） |
| `core_type` | enum | AIC/AIV/ANY/AICPU/HUB/GMATOMIC（V2 范围内主要使用 AIC 和 AIV） |
| `tileop` | string | TileOp 实现函数名（如 TileOp::Tadd） |
| `pipe` | list[2] | 使用的 Pipe 类型，格式为 `[start_pipe, end_pipe]`，对应 `TileOpCfg` 构造函数的 `pipeIdStart` 和 `pipeIdEnd`。可选值：`PIPE_S`(Scalar), `PIPE_V`(Vector), `PIPE_M`(Matrix/Cube), `PIPE_MTE1`/`PIPE_MTE2`/`PIPE_MTE3`(MTE), `PIPE_FIX`(FixPipe)。来源：`opcode.h` TileOpCfg 定义 |
| `dtype.*` | enum | supported/unsupported/partial |
| `operands.*.mem_type` | list | 允许的内存类型 |
| `operands.*.layout.type` | string | 排布格式要求 |
| `operands.*.layout.contiguous` | bool | 是否要求数据连续 |
| `operands.*.layout.max_dims` | int/null | 最大支持维度数 |
| `operands.*.stride.required` | bool | 是否必须有 stride |
| `operands.*.stride.constraints[].dim` | int | 有约束的维度索引 |
| `operands.*.stride.constraints[].constraint` | string | 约束描述 |
| `operands.*.stride.constraints[].reason` | string | 约束原因 |
| `operands.*.alignment` | struct | 地址对齐要求，结构为 `{address: int|null, scope: string|null}`（见 4.6 节） |
| `operands.*.shape.*` | varies | Shape 相关约束，支持以下规范模式（见下方 4.5 节） |
| `hardware.*` | varies | 硬件参数，公共常量见 _common.yaml |
| `tile_shape` | struct/null | TileShape 约束和推荐值，结构见 4.7 节。不需要用户设置的 OP（如 View）为 `null` |
| `tags` | list[str] | 对应 opcode.h 中的集合名 |
| `notes` | string | 经验备注，用 YAML 多行字符串 |
| `source` | string | 约束来源 |

**opcode_mapping.yaml Schema（Operation 级别）：**

| 字段路径 | 类型 | 说明 |
|----------|------|------|
| `description` | string | Operation 中文描述 |
| `category` | string | 对应 tileop_constraints 下的子目录名 |
| `opcodes[].opcode` | string | 该 Operation 可能用到的计算 OP_CODE |
| `opcodes[].condition` | string | 触发条件 |
| `opcodes[].core_type` | enum | 执行核心 |
| `pipeline[].stage` | string | 环节名称（copy_in, compute, copy_out, load, extract 等） |
| `pipeline[].opcode` | string | 该环节的 OP_CODE |
| `pipeline[].description` | string | 环节描述 |
| `pipeline[].direction` | string | 数据流向（GM → L1 → L0A 等） |
| `pipeline[].constraint_file` | string | 约束文件路径（相对于 tileop_constraints/） |
| `source` | string | 约束来源文件 |

### 4.5 Shape 约束规范模式

`operands.*.shape` 字段根据 OP 类别使用不同的规范模式：

**通用模式（Elementwise/Broadcast/Cast）：**
```yaml
shape:
  min_elements: 1            # 最少元素数
  max_elements_per_dim: null # 单维度最大元素数，null = 无限制
```

**矩阵模式（Cube Matmul/Conv）：**
```yaml
shape:
  min_m: 1                   # M 维度最小值
  min_k: 16                  # K 维度最小值（bf16 下一个 block）
  min_n: 1                   # N 维度最小值
  base_block_m: 16           # bf16 基础 block 大小
  base_block_k: 16
  base_block_n: 16
```

**一维模式（Reduce/Sort）：**
```yaml
shape:
  min_length: 1              # 最小长度
  max_length: null           # 最大长度，null = 无限制
  reduce_axis: 0             # reduce 的轴（仅 reduce 类）
```

当引入新的模式时，应在对应 YAML 文件中用注释说明。

### 4.6 Alignment 结构化字段

`operands.*.alignment` 使用结构化格式（而非自由文本），便于 AI 查询：

```yaml
alignment:
  address: 32    # 对齐字节数，null 表示无特殊要求
  scope: "首地址" # 对齐范围："首地址" / "每行首地址" / "所有维度" / null
```

示例：
```yaml
# 无特殊要求
alignment:
  address: null
  scope: null

# 32B 首地址对齐
alignment:
  address: 32
  scope: "首地址"

# 每行 32B 对齐
alignment:
  address: 32
  scope: "每行首地址"
```

### 4.7 TileShape 约束

TileShape 是用户设置的每次迭代处理的数据块大小。TileShape 设置之所以有约束，**本质上是因为底层 TileOp 的处理能力有限**——硬件的 mask 长度、repeat 次数、buffer 容量等参数直接决定了 TileShape 的合法范围。

因此，TileShape 约束作为 `tile_shape` 字段嵌入每个 OP_CODE 的 YAML 记录中，与 `hardware` 字段形成因果关系：`hardware` 描述 TileOp 的硬件能力边界，`tile_shape` 描述这些能力边界对用户设置 TileShape 的约束。

对于不需要用户设置 TileShape 的 OP（如 View、Assemble），该字段为 `null`。

#### 4.7.1 YAML 字段格式

```yaml
tile_shape:
  api: "set_vec_tile_shapes"           # 设置 API 名称
  api_params:                           # API 参数说明
    - name: "*args"
      type: "int"
      max_dims: 4
  constraints:                           # 约束列表（每条标注 TileOp 硬件根因）
    - rule: "末尾轴 * sizeof(dtype) 必须是 32B 对齐"
      severity: error
      tileop_cause: "UB buffer 按 32B block 访问，BLOCK_SIZE=32"
    - rule: "tile_shape 维度数 >= tensor 实际维度数"
      severity: error
      tileop_cause: "TileOp 按维度展开循环，维度不足会越界"
  recommended:                           # 推荐值（基于硬件参数推导）
    - scenario: "通用场景"
      value: "末尾轴向上取整到 32B 对齐"
      reasoning: "减少 unaligned 处理，最大化 mask 利用率"
```

**`tileop_cause` 字段说明**：每条约束标注其 TileOp 硬件根因，让使用者理解"为什么有这个约束"。例如：
- "末尾轴 32B 对齐" → 根因是 `BLOCK_SIZE=32`，vector 指令按 32B block 处理
- "单次 repeat 最多 255 次" → 根因是 `REPEAT_MAX=255`，硬件指令编码限制
- "L0A 容量限制" → 根因是 L0A 物理大小，决定了单次 matmul 的 M*K 上限

#### 4.7.2 Vector OP TileShape 约束

**设置 API:** `pypto.set_vec_tile_shapes(*args)`（最多 4 个维度）

**约束 → TileOp 硬件根因映射：**

| TileShape 约束 | TileOp 硬件根因 |
|----------------|----------------|
| 每个维度 > 0 | 循环步长不能为 0 |
| 末尾轴 * sizeof(dtype) % 32 == 0 | `BLOCK_SIZE=32`，vector 指令按 32B block 处理 |
| tile_shape 维度数 >= tensor 维度数 | TileOp 按维度展开循环 |
| 最大维度数 4 | API 参数限制 |
| 建议 (TensorShape/TileShape) * (1+输入数) < 18000 | 表达式表大小限制 |

**推荐值：**

| 场景 | 推荐设置 | 推理 |
|------|----------|------|
| 通用（bf16/fp16） | 末尾轴取整到 16 元素（32B），外轴保持 tensor 大小 | 末尾轴对齐到 BLOCK_SIZE=32，最大化 MASK_LEN=64 利用率 |
| 通用（fp32） | 末尾轴取整到 8 元素（32B），外轴保持 tensor 大小 | fp32 的 BLOCK_NELEM_B32=8 |
| 大批量（bf16） | 末尾轴=1024~4096 | 增大 tile 减少循环次数，但不超过 UB 容量 |
| 多输入 OP | 末尾轴 = max(各输入末尾轴) 向上取整到 32B | 广播展开后末尾轴取所有输入的最大值 |

#### 4.7.3 Cube OP TileShape 约束（Matmul）

**设置 API:** `pypto.set_cube_tile_shapes(m, k, n, enable_split_k=False)`

参数结构：`m=[mL0, mL1]`, `k=[kL0, kL1, kL1b]`, `n=[nL0, nL1]`

**约束 → TileOp 硬件根因映射：**

| TileShape 约束 | TileOp 硬件根因 |
|----------------|----------------|
| L0 <= L1 且 L1 % L0 == 0 | L1 → L0 是整块搬运，必须对齐 |
| kL0 * sizeof(dtype) % 32 == 0 | `BLOCK_SIZE=32`，L0A 按行存储需 32B 对齐 |
| nL0 * sizeof(dtype) % 32 == 0 | `BLOCK_SIZE=32`，L0B 按行存储需 32B 对齐 |
| NZ 外轴 16 元素对齐 | `BLOCK_CUBE_M_N=16`，NZ 格式的基本块大小 |
| L0A: CeilAlign(mL0,16) * CeilAlign(kL0,16) * sizeof(dtype) <= L0A_size | L0A 物理容量（A3: 1MB） |
| L0B: CeilAlign(nL0,16) * CeilAlign(kL0,16) * sizeof(dtype) <= L0B_size | L0B 物理容量（A3: 1MB） |
| L0C: CeilAlign(mL0,16) * CeilAlign(nL0,16) * 4 <= L0C_size | L0C 物理容量（A3: 2MB），输出固定 FP32 |
| INT8 时 M/K/N 对齐到 32 元素 | INT8 的 `BLOCK_NELEM_B8=32` |
| MX Matmul K 对齐到 64 元素 | MX 指令的内部块大小要求 |

**推荐值（A3 平台）：**

| 场景 | M [mL0, mL1] | K [kL0, kL1] | N [nL0, nL1] | 推理 |
|------|--------------|--------------|--------------|------|
| 通用 FP16 | [128, 128] | [64, 256] | [256, 256] | L0A=128*64*2=16KB, L0B=256*64*2=32KB, L0C=128*256*4=128KB，均在容量内 |
| 大 M 场景 | [256, 256] | [64, 256] | [128, 128] | 增大 mL0 提高计算密度，L0C=256*128*4=128KB |
| 大 K 场景 | [128, 128] | [128, 512] | [128, 128] | 增大 kL0/kL1 减少 K 维循环次数 |
| INT8 量化 | [128, 128] | [64, 256] | [128, 256] | 注意 32 元素对齐，L0A=128*64*1=8KB |
| 小模型 | [64, 64] | [32, 128] | [128, 128] | 节省 buffer，适合多 OP 共存场景 |

**选值原则：**
- mL0/kL0/nL0（L0 tile）：越大计算密度越高，但受 L0A/L0B/L0C 物理容量约束
- mL1/kL1/nL1（L1 tile）：控制 GM→L1 搬运粒度，需满足 L1 % L0 == 0
- L0A/L0B/L0C 三者需要平衡，避免任一溢出
- enable_split_k：K 较大且 L0 容量不足时启用

#### 4.7.4 Copy/Move OP TileShape 约束

**约束 → TileOp 硬件根因映射：**

| TileShape 约束 | TileOp 硬件根因 |
|----------------|----------------|
| CopyIn 末尾轴 C0 = 32/sizeof(dtype) | `BLOCK_SIZE=32`，MTE 按 32B block 搬运 |
| CopyIn 内轴 <= 65535 | ND 格式内轴最大值 |
| CopyOut N 轴 32B 对齐 | NZ 格式输出要求 |

#### 4.7.5 OP_CODE YAML 中的完整示例

```yaml
OP_ADD:
  # ... 其他字段 ...
  hardware:
    mask_len: 64
    repeat_max: 255
    block_size: 32
    elements_per_block:
      fp16: 16
      bf16: 16
      fp32: 8
  tile_shape:
    api: "set_vec_tile_shapes"
    constraints:
      - rule: "末尾轴 * sizeof(dtype) % 32 == 0"
        severity: error
        tileop_cause: "BLOCK_SIZE=32，vector 指令按 32B block 处理"
      - rule: "tile_shape 维度数 >= tensor 维度数"
        severity: error
        tileop_cause: "TileOp 按维度展开循环"
    recommended:
      - scenario: "通用 bf16"
        value: "末尾轴取整到 16 元素，外轴保持 tensor 大小"
        reasoning: "对齐 BLOCK_SIZE=32，最大化 MASK_LEN=64 利用率"

OP_A_MUL_B:
  # ... 其他字段 ...
  hardware:
    block_size: 32
    base_block_m: 16
    base_block_k: 16
    base_block_n: 16
  tile_shape:
    api: "set_cube_tile_shapes"
    constraints:
      - rule: "L0 <= L1 且 L1 % L0 == 0"
        severity: error
        tileop_cause: "L1 → L0 整块搬运，必须对齐"
      - rule: "kL0 * sizeof(dtype) % 32 == 0"
        severity: error
        tileop_cause: "L0A 按 32B block 行存储"
      - rule: "CeilAlign(mL0,16)*CeilAlign(kL0,16)*sizeof(dtype) <= L0A_size"
        severity: error
        tileop_cause: "L0A 物理容量限制"
    recommended:
      - scenario: "通用 FP16"
        value: "m=[128,128], k=[64,256], n=[256,256]"
        reasoning: "L0A=16KB, L0B=32KB, L0C=128KB，均在 A3 容量内"

OP_VIEW:
  # ... 其他字段 ...
  tile_shape: null  # View 不处理数据，无 TileOp 硬件约束
```

---

## 5. AI Skill 设计

### 5.1 基本信息

- **名称**：`pypto-op-capability-checker`
- **位置**：`.agents/skills/pypto-op/pypto-op-capability-checker/`
- **触发方式**：`/pypto-op-capability-checker <查询>`

### 5.2 查询模式

| 模式 | 示例 | 行为 |
|------|------|------|
| 按 OP 名 | `matmul bf16` | opcode_mapping → 找 pipeline 所有环节 → 逐个查 TileOp YAML → 输出全链路约束报告 |
| 按 OP_CODE | `OP_ADD` | 直接查 TileOp YAML → 输出报告 |
| 按能力 | `哪些 OP 支持 bf16` | 遍历 YAML 的 dtype.bf16 字段 → 筛选 → 输出列表 |
| 按类别 | `cube 类 OP 约束` | 查 cube/ 目录 → 汇总输出 |

> **网络级查询（如"attention 支持 bf16 吗"）**：V2 不在范围内。需要额外的网络→OP 分解映射数据，可作为后续扩展。

### 5.3 工作流程

```
用户输入
  ↓
1. 解析查询意图
   - 识别查询模式
   - 提取 dtype、shape 等上下文参数
  ↓
2. 定位数据文件
   - opcode_mapping.yaml → Operation → OP_CODE
   - tileop_constraints/<category>/<type>.yaml → 具体约束
  ↓
3. 读取并解析 YAML
  ↓
4. 生成结构化报告
  ↓
5. 信息不足时追问
```

### 5.4 Skill 文件

```
pypto-op-capability-checker/
├── SKILL.md                    # Skill 定义（触发条件、工作流程、查询指引）
└── references/
    ├── query_guide.md          # 查询模式说明和示例
    └── report_template.md      # 输出报告模板
```

Skill 不内嵌约束数据，引导 AI 读取 constraints/ 目录下的 YAML 文件。约束数据更新时无需修改 Skill。

---

## 6. 信息来源与维护

### 6.1 约束信息来源

| 来源 | 优先级 | 用途 |
|------|--------|------|
| `opcode.h` | 最高 | OP_CODE 定义、集合标签（UNSUPPORT_BF16_OPS 等） |
| `opcode.cpp` | 最高 | OP_CODE → core_type、mem_type、calc_type、TileOp 函数名 |
| `tileop/` 源码 | 高 | Layout/stride/alignment 约束 |
| `tileop_common.h` | 高 | 公共硬件常量 |
| 个人经验 | 中 | notes 字段，踩坑记录 |
| `docs/api/` | 低 | dtype/shape 约束补充 |

### 6.2 更新触发条件

- 新增/修改 OP_CODE
- TileOp 实现变更
- 发现新的约束条件（踩坑）
- 新增 Operation API

### 6.3 更新流程

1. 代码变更 → 更新对应 YAML 文件
2. commit message 标注 `docs(op): update constraint for XXX`

---

## 7. 实施计划

### 阶段一：框架搭建 + 核心约束

| 任务 | 产出 |
|------|------|
| 创建目录结构 | 完整 `.agents/skills/pypto-op/` 目录 |
| 编写 `_common.yaml` | 公共硬件常量 |
| 编写 `opcode_mapping.yaml` | 核心 Operation → OP_CODE 映射 |
| 编写核心 TileOp 约束 | ~8 个 YAML 文件 |
| 编写 `INDEX.md` | 总索引 |
| 编写 AI Skill | SKILL.md + references |

核心约束文件：
1. `vector/elementwise.yaml` — ADD, SUB, MUL, DIV, EXP, NEG, RELU, SQRT, ABS, LN...
2. `vector/elementwise_scalar.yaml` — ADDS, SUBS, MULS...
3. `vector/reduce.yaml` — ROWMAX, ROWSUM, ROWEXPMAX...
4. `vector/cast.yaml` — CAST, CONVERT
5. `cube/matmul.yaml` — A_MUL_B, A_MUL_BT, A_MULACC_B...
6. `cube/conv.yaml` — CONV, CONV_ADD
7. `move/copy_in.yaml` — COPY_IN, UB_COPY_IN, L1_COPY_IN...
8. `view/reshape.yaml` — RESHAPE, VIEW, ASSEMBLE

### 阶段二：补全剩余约束

| 任务 | 产出 |
|------|------|
| 补全 vector 类 | broadcast, gather_scatter, compare, logical, bitwise, sort |
| 补全 move 类 | copy_out, local_move, gather_load |
| 补全 view 类 | expand |
| 补全 misc 类 | duplicate, register, index, special |
| 补全 opcode_mapping.yaml | 所有 Operation |

### 阶段三：验证与优化

| 任务 | 产出 |
|------|------|
| 验证约束准确性 | 用代码交叉验证 |
| 补充 notes 字段 | 填入个人经验 |
| AI Skill 测试 | 多查询模式测试 |

---

## 附录 A：术语表

| 术语 | 说明 |
|------|------|
| Operation | Python API 层操作，如 `pypto.matmul` |
| OP_CODE | IR 层操作码，一个 Operation 可能对应多个 OP_CODE |
| TileOp | 处理 Tile 数据的底层实现函数 |
| Layout | 数据排布方式（NZ、ND、NC1HWC0 等） |
| Stride | 数据步长，各维度的元素间距 |
| Alignment | 地址对齐要求（32B 等） |
| UB | Unified Buffer，片上通用缓冲区 |
| L0A/L0B/L0C | Cube 单元的输入/输出缓冲区 |
| L1 | 片上一级缓冲区 |
| GM | Global Memory，全局内存 |
| MTE | Memory Transfer Engine，数据搬运引擎 |

## 附录 B：参考资料

- OP_CODE 定义：`framework/src/interface/operation/opcode.h`
- OP_CODE 注册：`framework/src/interface/operation/opcode.cpp`
- TileOp 实现：`framework/src/interface/tileop/`
- TileOp 公共常量：`framework/src/interface/tileop/tileop_common.h`
- Layout 定义：`framework/src/interface/tileop/utils/layout.h`
