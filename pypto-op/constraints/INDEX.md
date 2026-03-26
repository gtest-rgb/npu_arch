# Operation 能力边界约束知识库索引

> 本目录为 PyPTO Operation 层能力边界约束知识库，包含所有 TileOP 层 OP_CODE 的硬件约束规则。
> 所有 OP_CODE 总计 **196** 个（不含 _common.yaml 中的 4 个公共定义）。

## 目录结构总览

```
constraints/
├── INDEX.md                    # 本文件
├── opcode_mapping.yaml         # Operation API -> OP_CODE 执行链路映射
└── tileop_constraints/         # OP_CODE 约束规则（按功能域分类）
    ├── _common.yaml            # 公共定义：硬件常量、对齐规则、数据类型、标签集
    ├── cube/                   # 矩阵计算单元（AIC Core）
    ├── vector/                 # 向量计算单元（AIV Core）
    ├── move/                   # 数据搬运
    ├── view/                   # 视图与形状操作
    └── misc/                   # 其他杂项操作
```

## 各约束文件 OP_CODE 清单

### 公共定义

| 文件 | 顶级键数 | 说明 |
|------|---------|------|
| [`_common.yaml`](tileop_constraints/_common.yaml) | 4 | 硬件常量、对齐规则、数据类型约束、标签集（tag_sets） |

> `_common.yaml` 的 4 个顶级键为：`hardware_constants`、`alignment_rules`、`dtype_constraints`、`tag_sets`，不直接对应 OP_CODE，而是全局共享的基础定义。

### cube/ — 矩阵计算单元（AIC Core）

| 文件 | OP_CODE 数 | 主要 OP_CODE |
|------|-----------|-------------|
| [`conv.yaml`](tileop_constraints/cube/conv.yaml) | 7 | OP_CONV, OP_CONV_ADD, OP_CUBE_CONV_D2S, OP_CUBE_CONCAT_C, OP_L1_COPY_IN_CONV, OP_LOAD3D_CONV, OP_LOAD2D_CONV |
| [`matmul.yaml`](tileop_constraints/cube/matmul.yaml) | 6 | OP_A_MUL_B, OP_A_MULACC_B, OP_A_MUL_BT, OP_A_MULACC_BT, OP_AT_MUL_B, OP_AT_MUL_BT |

### vector/ — 向量计算单元（AIV Core）

| 文件 | OP_CODE 数 | 主要 OP_CODE |
|------|-----------|-------------|
| [`elementwise.yaml`](tileop_constraints/vector/elementwise.yaml) | 31 | OP_ADD, OP_SUB, OP_MUL, OP_DIV, OP_NEG, OP_EXP, OP_RSQRT, OP_RELU, OP_SQRT, OP_CEIL, OP_FLOOR, OP_TRUNC, OP_ROUND, OP_ABS, OP_LN, OP_RECIPROCAL, OP_LOG1P, OP_ISFINITE, OP_SIGN, OP_SIGNBIT, OP_MOD, OP_REM, OP_CLIP, OP_PAIRSUM, OP_PAIRMAX, OP_PAIRMIN, OP_PAIRPROD, OP_MAXIMUM, OP_MINIMUM, OP_GCD, OP_CEIL_DIV |
| [`elementwise_scalar.yaml`](tileop_constraints/vector/elementwise_scalar.yaml) | 13 | OP_ADDS, OP_SUBS, OP_MULS, OP_DIVS, OP_MAXS, OP_MINS, OP_LRELU, OP_S_ADDS, OP_S_SUBS, OP_S_MULS, OP_S_DIVS, OP_S_MAXS, OP_S_MINS |
| [`reduce.yaml`](tileop_constraints/vector/reduce.yaml) | 17 | OP_ROWMAX, OP_ROWSUM, OP_ROWEXPMAX, OP_ROWEXPSUM, OP_ROWSUMLINE, OP_ROWMAXLINE, OP_ROWMINLINE, OP_ROWMAX_SINGLE, OP_ROWMIN_SINGLE, OP_ROWSUM_SINGLE, OP_ROWPRODLINE, OP_ROWPROD_SINGLE, OP_ROWMAX_COMBINE_AXIS_SINGLE, OP_ROWSUM_COMBINE_AXIS_SINGLE 等 |
| [`sort.yaml`](tileop_constraints/vector/sort.yaml) | 9 | OP_SORT, OP_TOPK, OP_ARGSORT, OP_BITSORT, OP_MRGSORT, OP_COMPARE_SWAP, OP_MERGE, OP_EXTRACT, OP_SORT_UB |
| [`bitwise.yaml`](tileop_constraints/vector/bitwise.yaml) | 11 | OP_BITWISERIGHTSHIFT, OP_BITWISELEFTSHIFT, OP_BITWISEAND, OP_BITWISEOR, OP_BITWISEXOR, OP_BITWISENOT 及标量版本 |
| [`broadcast.yaml`](tileop_constraints/vector/broadcast.yaml) | 6 | OP_ADD_BRC, OP_SUB_BRC, OP_MUL_BRC, OP_DIV_BRC, OP_MAX_BRC, OP_MIN_BRC |
| [`cast.yaml`](tileop_constraints/vector/cast.yaml) | 2 | OP_CAST, OP_CONVERT |
| [`compare.yaml`](tileop_constraints/vector/compare.yaml) | 2 | OP_CMP, OP_CMPS |
| [`logical.yaml`](tileop_constraints/vector/logical.yaml) | 6 | OP_LOGICALNOT, OP_LOGICALAND, OP_WHERE_TT, OP_WHERE_TS, OP_WHERE_ST, OP_WHERE_SS |
| [`gather_scatter.yaml`](tileop_constraints/vector/gather_scatter.yaml) | 9 | OP_GATHER, OP_GATHER_FROM_UB, OP_GATHER_ELEMENT, OP_GATHER_MASK, OP_GATHER_MASK_BUILDIN, OP_SCATTER, OP_SCATTER_ELEMENT, OP_SCATTER_UPDATE, OP_SCATTER_SCALAR |

### move/ — 数据搬运

| 文件 | OP_CODE 数 | 主要 OP_CODE |
|------|-----------|-------------|
| [`copy_in.yaml`](tileop_constraints/move/copy_in.yaml) | 16 | OP_COPY_IN, OP_UB_COPY_IN, OP_L1_COPY_IN, OP_L1_TO_L0A, OP_L1_TO_L0B, OP_L1_TO_BT, OP_TRANSPOSE_MOVEIN 等 |
| [`copy_out.yaml`](tileop_constraints/move/copy_out.yaml) | 20 | OP_COPY_OUT, OP_UB_COPY_OUT, OP_L0C_COPY_OUT, OP_L1_COPY_OUT, OP_TRANSPOSE_MOVEOUT, OP_L1_TO_FIX, OP_FIX_TO_L1 等 |
| [`local_move.yaml`](tileop_constraints/move/local_move.yaml) | 7 | OP_L1_COPY_UB, OP_L0C_COPY_UB, OP_UB_COPY_L1, OP_UB_COPY_L1_ND, OP_COPY_UB_TO_UB, OP_L1_TO_L1, OP_UB_COPY_ND2NZ |
| [`gather_load.yaml`](tileop_constraints/move/gather_load.yaml) | 1 | OP_LOAD |

### view/ — 视图与形状操作

| 文件 | OP_CODE 数 | 主要 OP_CODE |
|------|-----------|-------------|
| [`view.yaml`](tileop_constraints/view/view.yaml) | 2 | OP_VIEW, OP_VIEW_TYPE |
| [`reshape.yaml`](tileop_constraints/view/reshape.yaml) | 3 | OP_RESHAPE, OP_RESHAPE_COPY_IN, OP_RESHAPE_COPY_OUT |
| [`expand.yaml`](tileop_constraints/view/expand.yaml) | 2 | OP_EXPAND, OP_EXPANDEXPDIF |
| [`assemble.yaml`](tileop_constraints/view/assemble.yaml) | 2 | OP_ASSEMBLE, OP_ASSEMBLE_SSA |

### misc/ — 杂项操作

| 文件 | OP_CODE 数 | 主要 OP_CODE |
|------|-----------|-------------|
| [`special.yaml`](tileop_constraints/misc/special.yaml) | 14 | OP_CONCAT, OP_CUM_SUM, OP_ONEHOT, OP_PAD, OP_COMPACT, OP_RANGE, OP_HYPOT, OP_PRELU, OP_POW, OP_TRIUL, OP_BRCB, OP_MODS, OP_REMS, OP_REMRS |
| [`index.yaml`](tileop_constraints/misc/index.yaml) | 3 | OP_INDEX_OUTCAST, OP_INDEX_PUT, OP_INDEX_ADD |
| [`duplicate.yaml`](tileop_constraints/misc/duplicate.yaml) | 2 | OP_DUPLICATE, OP_VEC_DUP |
| [`register.yaml`](tileop_constraints/misc/register.yaml) | 1 | OP_REGISTER_COPY |

## 快速查询表：常见 Operation -> 约束文件

> 下表列出常用高层 Operation API 及其对应的 OP_CODE 和约束文件，方便快速定位。

| Operation | 核心 OP_CODE | 计算约束文件 | 搬运约束文件 | 分类 |
|-----------|-------------|-------------|-------------|------|
| `add` | OP_ADD / OP_ADD_BRC / OP_ADDS | vector/elementwise.yaml / vector/broadcast.yaml / vector/elementwise_scalar.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `sub` | OP_SUB / OP_SUB_BRC / OP_SUBS | vector/elementwise.yaml / vector/broadcast.yaml / vector/elementwise_scalar.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `mul` | OP_MUL / OP_MUL_BRC / OP_MULS | vector/elementwise.yaml / vector/broadcast.yaml / vector/elementwise_scalar.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `div` | OP_DIV / OP_DIV_BRC / OP_DIVS | vector/elementwise.yaml / vector/broadcast.yaml / vector/elementwise_scalar.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `neg` | OP_NEG | vector/elementwise.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `exp` | OP_EXP | vector/elementwise.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `sqrt` | OP_SQRT | vector/elementwise.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `rsqrt` | OP_RSQRT | vector/elementwise.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `relu` | OP_RELU | vector/elementwise.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `cast` | OP_CAST / OP_CONVERT | vector/cast.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `sum` | OP_ROWSUM / OP_ROWSUM_SINGLE | vector/reduce.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `max` (amax) | OP_ROWMAX / OP_ROWMAX_SINGLE | vector/reduce.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `min` (amin) | OP_ROWMAXLINE / OP_ROWMIN_SINGLE | vector/reduce.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `matmul` | OP_A_MUL_B / OP_A_MULACC_B / OP_A_MUL_BT 等 | cube/matmul.yaml | move/copy_in.yaml, move/copy_out.yaml | cube |
| `scaled_mm` | OP_A_MUL_B (MX) | cube/matmul.yaml | move/copy_in.yaml, move/copy_out.yaml | cube |
| `conv` | OP_CONV | cube/conv.yaml | move/copy_in.yaml, move/copy_out.yaml | cube |
| `greater/less/equal` | OP_CMP / OP_CMPS | vector/compare.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `where` | OP_WHERE_TT / OP_WHERE_TS / OP_WHERE_ST / OP_WHERE_SS | vector/logical.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `gather` | OP_GATHER_ELEMENT | vector/gather_scatter.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `scatter` | OP_SCATTER_ELEMENT / OP_SCATTER | vector/gather_scatter.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `index_add` | OP_INDEX_ADD | misc/index.yaml | move/copy_in.yaml, move/copy_out.yaml | misc |
| `index_put` | OP_INDEX_PUT | misc/index.yaml | move/copy_in.yaml, move/copy_out.yaml | misc |
| `topk` | OP_TOPK_SORT / OP_TOPK_MERGE / OP_TOPK_EXTRACT | vector/sort.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `argsort` | OP_BITSORT / OP_MRGSORT | vector/sort.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |
| `concat` | OP_CONCAT | misc/special.yaml | move/copy_in.yaml, move/copy_out.yaml | misc |
| `pad` | OP_PAD | misc/special.yaml | move/copy_in.yaml, move/copy_out.yaml | misc |
| `transpose` | OP_VIEW | view/view.yaml | -- | view |
| `reshape` | OP_RESHAPE | view/reshape.yaml | -- | view |
| `expand` | OP_EXPAND | view/expand.yaml | -- | view |
| `bitwise_and/or/xor` | OP_BITWISEAND / OP_BITWISEOR / OP_BITWISEXOR | vector/bitwise.yaml | move/copy_in.yaml, move/copy_out.yaml | vector |

> 完整的 Operation -> OP_CODE -> 约束文件映射请参阅 [`opcode_mapping.yaml`](opcode_mapping.yaml)。

## 约束维度说明

每个 OP_CODE 的约束规则通常包含以下维度：

### 1. Layout（存储格式）

定义 Tensor 数据在存储介质上的排列方式。

- **ND (No Dimmer)**: 通用布局，数据按行优先连续存储
- **NZ (Non-Zero)**: 非零压缩布局，16x16 分块转置存储，AIC Core 矩阵计算的标准格式
- **FRACTAL_Z**: 分形 Z 布局，Cube 指令要求的权重/特征图存储格式
- **NC1HWC0**: 5D 布局，常用于卷积的特征图和权重

### 2. Stride（步长）

定义相邻元素在内存中的地址偏移量。

- 对齐到 `BLOCK_SIZE`（32 Byte）的整数倍
- 某些 OP_CODE 要求连续存储（stride = element_size）
- 广播场景下允许 stride 为 0（同一数据被重复使用）

### 3. Address Alignment（地址对齐）

定义起始地址的对齐要求。

- **GM（Global Memory）**: 32 Byte 对齐
- **L1（Level 1 Buffer）**: 32 Byte 对齐
- **UB（Unified Buffer）**: 32 Byte 对齐
- **L0A / L0B / L0C**: 32 Byte 对齐

> 完整对齐规则见 [`_common.yaml`](tileop_constraints/_common.yaml) 中的 `alignment_rules`。

### 4. Hardware（硬件资源限制）

定义硬件计算资源的约束条件。

- **BLOCK_SIZE**: 32 Byte（一次向量计算的基本单位）
- **REPEAT_MAX**: 255（单条指令最大重复次数）
- **MASK_LEN**: 64（掩码位长度）
- **数据类型支持**: fp16, bf16, fp32, int8 等
- **数据类型限制**: 部分运算不支持 bf16（如 OP_EXP, OP_RSQRT），部分不支持 fp16（如 OP_MOD）

> 完整硬件常量见 [`_common.yaml`](tileop_constraints/_common.yaml) 中的 `hardware_constants` 和 `dtype_constraints`。

### 5. TileShape（分块形状）

定义矩阵/向量计算中分块的形状约束。

- 矩阵乘法的 M/K/N 维度通常需满足 16 的整数倍
- Cube 指令要求输入为 FractalZ/NZ 等特定布局的 Tile
- 向量指令的元素数需为 8（fp32）或 16（fp16/bf16）的整数倍
- 支持 dynamic_unaligned 的 OP_CODE 允许非对齐的元素数

> 完整支持动态非对齐的 OP_CODE 列表见 [`_common.yaml`](tileop_constraints/_common.yaml) 中的 `tag_sets.SUPPORT_DYNAMIC_UNALIGNED_OPS`。

## 相关文件

| 文件 | 说明 |
|------|------|
| [`opcode_mapping.yaml`](opcode_mapping.yaml) | 高层 Operation API 到 OP_CODE 执行链路的完整映射 |
| [`_common.yaml`](tileop_constraints/_common.yaml) | 公共硬件常量、对齐规则、数据类型约束、标签集 |
