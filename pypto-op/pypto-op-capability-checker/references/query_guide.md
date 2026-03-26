# 查询模式指南

本指南描述 Operation Capability Checker 的四种查询模式的详细操作步骤和示例。

---

## 查询模式一：按OP名查询

### 适用场景

用户通过高层 Operation API 名称进行查询，例如：
- "matmul 支持 bf16 吗"
- "add 的执行链路是什么"
- "conv 有哪些硬件约束"

### 检索步骤

```
1. 解析 OP 名称
   └─ 从用户查询中提取 Operation 名称（如 matmul、add、conv）

2. 查找 opcode_mapping.yaml
   └─ 定位: .agents/skills/pypto-op/constraints/opcode_mapping.yaml
   └─ 以 OP 名称为 key，获取该 Operation 的完整条目

3. 提取关键信息
   ├─ description: 中文描述
   ├─ category: 功能域分类（cube/vector/move/view/misc）
   ├─ opcodes: 该 Operation 可能使用的所有 OP_CODE（含触发条件和计算核心类型）
   ├─ pipeline: 完整执行链路（每个 stage 含 opcode、description、direction、constraint_file）
   └─ source: Python API 源文件路径

4. 逐阶段读取约束
   ├─ 遍历 pipeline 中的每个 stage
   ├─ 根据 constraint_file 路径拼接完整路径:
   │   .agents/skills/pypto-op/constraints/tileop_constraints/{constraint_file}
   └─ 在 YAML 中查找对应 OP_CODE 的约束规则

5. 读取公共定义
   └─ .agents/skills/pypto-op/constraints/tileop_constraints/_common.yaml
   └─ 获取 hardware_constants、alignment_rules、dtype_constraints、tag_sets

6. 生成报告
   └─ 按 report_template.md 格式输出
```

### 示例

**查询：** "matmul 支持 bf16 吗"

**检索过程：**

1. 在 `opcode_mapping.yaml` 中找到 `matmul` 条目
2. category = cube
3. 核心计算 OP_CODE: OP_A_MUL_B, OP_A_MULACC_B, OP_A_MUL_BT, OP_A_MULACC_BT, OP_AT_MUL_B, OP_AT_MUL_BT
4. Pipeline 包含 7 个 stage:
   - copy_in_a: OP_COPY_IN (GM -> L1)
   - copy_in_b: OP_COPY_IN (GM -> L1)
   - load_a: OP_L1_TO_L0A (L1 -> L0A)
   - load_b: OP_L1_TO_L0B (L1 -> L0B)
   - mmad: OP_A_MUL_B (L0A,L0B -> L0C) -- constraint_file: cube/matmul.yaml
   - extract_c: OP_L0C_COPY_OUT (L0C -> L1)
   - copy_out: OP_COPY_OUT (L1 -> GM)
5. 读取 `cube/matmul.yaml` 中 OP_A_MUL_B 的 dtype 约束
6. 读取 `_common.yaml` 获取公共数据类型约束

**输出要点：**
- matmul（OP_A_MUL_B）支持 bf16
- 输入 Layout: NZ 或 FRACTAL_Z
- M/K/N 维度需满足 16 的整数倍
- 需要调用 set_cube_tile_shapes 设置 TileShape

### 注意事项

- 同一 Operation 可能因条件不同使用不同 OP_CODE。例如 `add` 可能使用 OP_ADD（两个 Tensor）、OP_ADD_BRC（broadcast 标量）、OP_ADDS（立即数标量），各 OP_CODE 的约束可能不同
- pipeline 是逻辑视图，实际代码生成可能融合或省略部分环节
- 如果用户没有指定具体条件，应报告所有可能的 OP_CODE 及各自约束

---

## 查询模式二：按OP_CODE查询

### 适用场景

用户直接查询 OP_CODE 级别的约束，例如：
- "OP_ADD 的约束"
- "OP_A_MUL_B 支持 NZ layout 吗"
- "OP_CONV 的地址对齐要求"

### 检索步骤

```
1. 解析 OP_CODE 名称
   └─ 从用户查询中提取 OP_CODE 名称（如 OP_ADD、OP_A_MUL_B）

2. 定位约束文件
   ├─ 方式A: 参考 INDEX.md 的"各约束文件 OP_CODE 清单"
   ├─ 方式B: 在 opcode_mapping.yaml 中反向查找（搜索 opcodes.opcode 字段）
   └─ 方式C: 遍历 tileop_constraints/ 下的 YAML 文件搜索

3. 读取约束规则
   └─ 在目标 YAML 文件中以 OP_CODE 名称为 key，读取约束规则

4. 读取公共定义
   └─ _common.yaml

5. 生成报告
   └─ 按 report_template.md 格式输出
```

### OP_CODE 快速定位表

| OP_CODE | 约束文件 | 分类 |
|---------|----------|------|
| OP_ADD, OP_SUB, OP_MUL, OP_DIV, OP_NEG, OP_EXP, ... | vector/elementwise.yaml | vector |
| OP_ADDS, OP_SUBS, OP_MULS, OP_DIVS, OP_MAXS, OP_MINS, ... | vector/elementwise_scalar.yaml | vector |
| OP_A_MUL_B, OP_A_MULACC_B, OP_A_MUL_BT, ... | cube/matmul.yaml | cube |
| OP_CONV, OP_CONV_ADD, ... | cube/conv.yaml | cube |
| OP_COPY_IN, OP_UB_COPY_IN, OP_L1_TO_L0A, ... | move/copy_in.yaml | move |
| OP_COPY_OUT, OP_L0C_COPY_OUT, OP_L1_COPY_OUT, ... | move/copy_out.yaml | move |
| OP_VIEW, OP_VIEW_TYPE | view/view.yaml | view |
| OP_RESHAPE, OP_RESHAPE_COPY_IN, OP_RESHAPE_COPY_OUT | view/reshape.yaml | view |
| OP_EXPAND, OP_EXPANDEXPDIF | view/expand.yaml | view |
| OP_ASSEMBLE, OP_ASSEMBLE_SSA | view/assemble.yaml | view |
| OP_CAST, OP_CONVERT | vector/cast.yaml | vector |
| OP_CMP, OP_CMPS | vector/compare.yaml | vector |
| OP_ROWSUM, OP_ROWMAX, OP_ROWEXPMAX, ... | vector/reduce.yaml | vector |
| OP_SORT, OP_TOPK, OP_ARGSORT, OP_BITSORT, ... | vector/sort.yaml | vector |
| OP_CONCAT, OP_CUM_SUM, OP_ONEHOT, OP_PAD, ... | misc/special.yaml | misc |
| OP_GATHER, OP_SCATTER, OP_GATHER_ELEMENT, ... | vector/gather_scatter.yaml | vector |
| OP_LOAD | move/gather_load.yaml | move |

> 完整清单见 `constraints/INDEX.md`。

### 示例

**查询：** "OP_EXP 的约束"

**检索过程：**

1. 定位: OP_EXP 在 `vector/elementwise.yaml`
2. 读取 OP_EXP 条目的约束规则
3. 读取 `_common.yaml` 公共定义

**输出要点：**
- 数据类型: 仅支持 fp16、fp32（不支持 bf16）
- Layout: ND
- 地址对齐: UB 32B
- 标签: unary_op, elementwise_op, no_bf16（如适用）

---

## 查询模式三：按能力维度查询

### 适用场景

用户按特定约束维度跨 OP_CODE 检索，例如：
- "哪些 OP_CODE 不支持 bf16"
- "哪些 OP_CODE 支持 dynamic_unaligned"
- "所有支持 NZ layout 的 OP_CODE"
- "哪些 OP_CODE 是 binary_op"

### 检索步骤

```
1. 解析查询维度
   └─ 从用户查询中提取维度关键词（如 bf16、dynamic_unaligned、NZ、binary_op）

2. 读取公共定义
   └─ _common.yaml
   ├─ tag_sets: 预定义的 OP_CODE 集合（如 BINARY_OPS、SUPPORT_DYNAMIC_UNALIGNED_OPS）
   └─ dtype_constraints: 数据类型基本参数

3. 遍历分类约束文件
   ├─ cube/: conv.yaml, matmul.yaml
   ├─ vector/: elementwise.yaml, elementwise_scalar.yaml, reduce.yaml, sort.yaml,
   │          bitwise.yaml, broadcast.yaml, cast.yaml, compare.yaml, logical.yaml,
   │          gather_scatter.yaml
   ├─ move/: copy_in.yaml, copy_out.yaml, local_move.yaml, gather_load.yaml
   ├─ view/: view.yaml, reshape.yaml, expand.yaml, assemble.yaml
   └─ misc/: special.yaml, index.yaml, duplicate.yaml, register.yaml

4. 按维度过滤
   └─ 对每个 YAML 文件中的每个 OP_CODE，检查目标维度是否匹配

5. 汇总输出
   └─ 列出匹配的 OP_CODE 及其所在的约束文件
```

### 常见维度查询

#### 数据类型查询

| 查询 | 检索方法 |
|------|----------|
| 哪些 OP_CODE 支持 bf16 | 遍历所有 YAML，筛选 supported_dtypes 包含 bf16 的条目 |
| 哪些 OP_CODE 不支持 bf16 | 遍历所有 YAML，筛选 supported_dtypes 不包含 bf16 的条目 |
| 哪些 OP_CODE 仅支持 fp16/fp32 | 遍历所有 YAML，筛选 supported_dtypes 不包含 bf16 的条目 |

**典型不支持 bf16 的 OP_CODE：** OP_EXP、OP_RSQRT、OP_RELU、OP_EXP2、OP_EXPM1、OP_LN、OP_LOG1P、OP_HYPOT、OP_ISFINITE、OP_POW、OP_MOD、OP_REM 等（具体以各 YAML 文件为准）

#### 标签集查询

| 查询 | 检索方法 |
|------|----------|
| dynamic_unaligned OP | 查找 `_common.yaml` 的 `tag_sets.SUPPORT_DYNAMIC_UNALIGNED_OPS` |
| binary_op OP | 查找 `_common.yaml` 的 `tag_sets.BINARY_OPS` |
| unary_op OP | 查找 `_common.yaml` 的 `tag_sets.UNARY_OPS` |
| reduction_op OP | 查找 `_common.yaml` 的 `tag_sets.REDUCTION_OPS` |

#### Layout 查询

| 查询 | 检索方法 |
|------|----------|
| 支持 NZ layout 的 OP | 遍历所有 YAML，筛选 supported_layouts 包含 NZ 的条目 |
| 需要 FRACTAL_Z 的 OP | 遍历所有 YAML，筛选 required_layouts 包含 FRACTAL_Z 的条目 |

### 示例

**查询：** "哪些 OP_CODE 不支持 bf16"

**检索过程：**

1. 遍历 tileop_constraints/ 下所有 YAML 文件
2. 对每个 OP_CODE 条目，检查 `supported_dtypes` 字段
3. 收集不含 bf16 的 OP_CODE

**输出要点：**
- OP_EXP (vector/elementwise.yaml) -- 仅 fp16, fp32
- OP_RSQRT (vector/elementwise.yaml) -- 仅 fp16, fp32
- OP_MOD (vector/elementwise.yaml) -- 仅 fp16, fp32
- ...（完整列表）

---

## 查询模式四：按OP类别查询

### 适用场景

用户按功能域分类查询，例如：
- "cube 类 OP 有哪些约束"
- "vector 类所有 OP_CODE 列表"
- "move 类的共性约束是什么"

### 检索步骤

```
1. 解析类别名称
   └─ cube / vector / move / view / misc

2. 进入对应目录
   ├─ cube/: 矩阵计算单元（AIC Core）
   ├─ vector/: 向量计算单元（AIV Core）
   ├─ move/: 数据搬运
   ├─ view/: 视图与形状操作
   └─ misc/: 杂项操作

3. 读取该目录下所有 YAML 文件

4. 汇总分析
   ├─ 列出该类别下所有 OP_CODE
   ├─ 提取共性约束（如 Layout、对齐要求等）
   └─ 标注特殊约束（某些 OP_CODE 独有的约束）

5. 生成报告
```

### 类别概览

| 类别 | 说明 | OP_CODE 数量 | 子目录 |
|------|------|-------------|--------|
| cube | 矩阵计算单元（AIC Core） | 13 | conv.yaml, matmul.yaml |
| vector | 向量计算单元（AIV Core） | 107 | elementwise.yaml, elementwise_scalar.yaml, reduce.yaml, sort.yaml, bitwise.yaml, broadcast.yaml, cast.yaml, compare.yaml, logical.yaml, gather_scatter.yaml |
| move | 数据搬运 | 44 | copy_in.yaml, copy_out.yaml, local_move.yaml, gather_load.yaml |
| view | 视图与形状操作 | 9 | view.yaml, reshape.yaml, expand.yaml, assemble.yaml |
| misc | 杂项操作 | 23 | special.yaml, index.yaml, duplicate.yaml, register.yaml |

### 各类别共性特征

#### cube 类
- 计算核心: AIC Core
- Layout: NZ / FRACTAL_Z
- 存储层级: L0A, L0B, L0C, L1
- 关键约束: M/K/N 维度对齐、TileShape 设置
- 典型 OP: OP_A_MUL_B, OP_CONV

#### vector 类
- 计算核心: AIV Core
- Layout: ND
- 存储层级: UB（Unified Buffer）
- 关键约束: 元素数对齐（fp32: 8的倍数, fp16/bf16: 16的倍数）、REPEAT_MAX
- 典型 OP: OP_ADD, OP_MUL, OP_ROWSUM, OP_CAST

#### move 类
- 计算核心: 无（纯搬运）
- Layout: 取决于源和目标存储层级
- 存储层级: GM, L1, UB, L0A, L0B, L0C, BT, FIX
- 关键约束: 地址对齐（32B）、数据类型一致性
- 典型 OP: OP_COPY_IN, OP_COPY_OUT, OP_L1_TO_L0A

#### view 类
- 计算核心: 无（元数据操作）
- Layout: 不改变数据布局，仅改变视图映射
- 存储层级: N/A（不搬运数据）
- 关键约束: 形状兼容性、offset 对齐
- 典型 OP: OP_VIEW, OP_RESHAPE, OP_EXPAND, OP_ASSEMBLE

#### misc 类
- 计算核心: 混合（AIV Core 为主）
- Layout: ND
- 存储层级: UB
- 关键约束: 因 OP 而异
- 典型 OP: OP_CONCAT, OP_PAD, OP_ONEHOT, OP_RANGE

### 示例

**查询：** "cube 类 OP 有哪些约束"

**检索过程：**

1. 进入 `tileop_constraints/cube/` 目录
2. 读取 `conv.yaml`（7 个 OP_CODE）和 `matmul.yaml`（6 个 OP_CODE）
3. 分析共性约束和特殊约束

**输出要点：**
- 总计 13 个 OP_CODE
- 共性约束: NZ/FRACTAL_Z Layout、L0A/L0B/L0C 存储层级
- 各 OP_CODE 的独立约束详见各 YAML 文件

---

## 查询模式选择指南

当用户查询不明确时，按以下优先级判断查询模式：

1. **包含 OP_CODE 前缀（OP_）** -- 按OP_CODE查询
2. **包含已知 Operation 名称** -- 按OP名查询
3. **包含功能域名称** -- 按OP类别查询
4. **包含约束维度关键词** -- 按能力维度查询
5. **无法判断** -- 追问用户意图
