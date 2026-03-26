# OP 能力边界查询报告模板

## Operation 能力边界查询报告

---

### 基本信息

| 维度 | 内容 |
|------|------|
| Operation 名称 | [OP名，如 matmul] |
| OP_CODE | [核心 OP_CODE，如 OP_A_MUL_B] |
| 类别 | [cube / vector / move / view / misc] |
| 计算核心 | [AIC / AIV / ANY] |
| Python API | [源文件路径，如 python/pypto/op/matmul.py] |

> 如果同一 Operation 有多个可能的 OP_CODE，列出所有变体：

| OP_CODE | 触发条件 | 计算核心 |
|---------|----------|----------|
| [OP_CODE_1] | [条件描述] | [AIC/AIV] |
| [OP_CODE_2] | [条件描述] | [AIC/AIV] |

---

### Pipeline 概览

> 仅在按OP名查询时展示此节。

| 阶段 | OP_CODE | 描述 | 数据流向 | 约束文件 |
|------|---------|------|----------|----------|
| [stage_1] | [opcode] | [description] | [direction] | [constraint_file] |
| [stage_2] | [opcode] | [description] | [direction] | [constraint_file] |
| ... | ... | ... | ... | ... |

**Pipeline 说明：**
> [一句话说明 Pipeline 的数据流向和关键特征，如 matmul 的 GM->L1->L0A/L0B->L0C->L1->GM 链路]

---

### 约束详情

> 按阶段或按约束维度组织。按OP名查询时按阶段组织，按OP_CODE查询时按维度组织。

#### [阶段名称 / 约束维度名称]

**Layout（存储格式）**

| 参数 | 约束值 | 说明 |
|------|--------|------|
| supported_layouts | [ND / NZ / FRACTAL_Z / NC1HWC0] | 支持的存储格式 |
| required_layouts | [如 NZ] | 必须的存储格式 |
| input_layout | [如 NZ] | 输入 Layout 要求 |
| output_layout | [如 ND] | 输出 Layout 要求 |

**Stride（步长对齐）**

| 参数 | 约束值 | 说明 |
|------|--------|------|
| stride_alignment | [如 32B / BLOCK_SIZE] | 步长对齐要求 |
| contiguous_required | [true / false] | 是否要求连续存储 |
| broadcast_stride_zero | [true / false] | 是否允许 stride=0（广播） |

**Address Alignment（地址对齐）**

| 存储层级 | 对齐要求 | 说明 |
|----------|----------|------|
| GM | [32B] | 全局内存 |
| L1 | [32B] | Level 1 缓冲 |
| UB | [32B] | 统一缓冲区 |
| L0A | [32B] | Matrix A 输入缓冲 |
| L0B | [32B] | Matrix B 输入缓冲 |
| L0C | [32B] | Matrix C 输出缓冲 |

> 注：所有存储层级的默认对齐为 32B，来源于 `_common.yaml` 的 `alignment_rules`。

**Hardware（硬件参数）**

| 参数 | 值 | 说明 |
|------|-----|------|
| BLOCK_SIZE | 32 Byte | 一次向量计算的基本单位 |
| MASK_LEN | 64 | 掩码位长度 |
| REPEAT_MAX | 255 | 单条指令最大重复次数 |
| [其他参数] | [值] | [说明] |

**TileShape（分块形状）**

| 维度 | 约束值 | 说明 |
|------|--------|------|
| M | [如 16 的整数倍] | 行维度 |
| K | [如 16 的整数倍] | 公共维度 |
| N | [如 16 的整数倍] | 列维度 |
| element_count | [如 8/16 的整数倍] | 元素数对齐 |
| dynamic_unaligned | [supported / not supported] | 是否支持动态非对齐 |

**Data Type（数据类型）**

| 数据类型 | 元素大小 | BLOCK_NELEM | 支持情况 |
|----------|----------|-------------|----------|
| fp16 | 2 Byte | 16 | [支持/不支持] |
| bf16 | 2 Byte | 16 | [支持/不支持] |
| fp32 | 4 Byte | 8 | [支持/不支持] |
| int8 | 1 Byte | 32 | [支持/不支持] |

---

### 标签

> 关联 `_common.yaml` 中 `tag_sets` 的标签。

| 标签集 | 是否匹配 | 说明 |
|--------|----------|------|
| [标签名，如 BINARY_OPS] | [是/否] | [标签说明] |
| [标签名，如 SUPPORT_DYNAMIC_UNALIGNED_OPS] | [是/否] | [标签说明] |
| [标签名，如 UNARY_OPS] | [是/否] | [标签说明] |

---

### 备注

> 特殊注意事项、已知限制、使用建议等。

- [注意事项1]
- [注意事项2]

---

### 来源

| 约束维度 | 来源文件 |
|----------|----------|
| Operation 映射 | `.agents/skills/pypto-op/constraints/opcode_mapping.yaml` |
| OP_CODE 约束 | `.agents/skills/pypto-op/constraints/tileop_constraints/[分类]/[文件].yaml` |
| 公共定义 | `.agents/skills/pypto-op/constraints/tileop_constraints/_common.yaml` |
| 知识库索引 | `.agents/skills/pypto-op/constraints/INDEX.md` |

---

## 输出格式说明

### 约束严格程度标记

| 标记 | 含义 |
|------|------|
| **硬约束** | 必须满足，否则功能失败或产生未定义行为 |
| **软约束** | 建议满足，超出后可能降级处理 |
| **性能约束** | 影响性能但不影响正确性 |

### 引用格式

所有约束来源必须使用以下格式引用：
```
来源：`.agents/skills/pypto-op/constraints/tileop_constraints/[分类]/[文件].yaml`
```

例如：
```
来源：`.agents/skills/pypto-op/constraints/tileop_constraints/cube/matmul.yaml`
来源：`.agents/skills/pypto-op/constraints/tileop_constraints/_common.yaml`
```

### 查询信息

- 查询时间：[YYYY-MM-DD HH:MM]
- 知识库版本：[版本号或 commit hash]
- 查询模式：[按OP名 / 按OP_CODE / 按能力维度 / 按OP类别]
