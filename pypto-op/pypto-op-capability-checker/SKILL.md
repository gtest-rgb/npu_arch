---
name: pypto-op-capability-checker
description: Operation模块能力边界判断工具。用于查询OP/OP_CODE的能力约束，包括Layout、Stride对齐、地址对齐、硬件参数和TileShape约束。支持按OP名、OP_CODE、能力维度、OP类别四种查询模式。
---

# PyPTO Operation Capability Checker Skill

## 概述

本技能用于查询 PyPTO Operation 层的能力边界约束，输出指定 OP 或 OP_CODE 的硬件约束规则，包括 Layout、Stride 对齐、地址对齐、硬件参数限制和 TileShape 约束。

知识库覆盖全部 **196** 个 OP_CODE 的约束规则，按功能域分类存储。

### 使用场景

- 代码评审：评审自定义算子或优化方案时，快速查询 OP_CODE 的硬件约束
- 设计讨论：讨论新 Operation 实现方案时，确认底层 OP_CODE 的能力边界
- 对外技术支持：回答用户/开发者关于 Operation 支持范围的问题

### 触发机制

手动调用：`/pypto-op-capability-checker <查询内容>`

### 功能概述

1. 按OP名查询：通过高层 Operation API 名称（如 matmul、add）查询完整的 OP_CODE 执行链路及各阶段约束
2. 按OP_CODE查询：直接查询指定 OP_CODE（如 OP_ADD、OP_A_MUL_B）的硬件约束
3. 按能力维度查询：按特定约束维度（如 dtype、layout、dynamic_unaligned）跨 OP_CODE 检索
4. 按OP类别查询：按功能域（cube/vector/move/view/misc）查询类别级别的约束概览

## 工作流程

按照以下流程执行：

### 1. 场景理解阶段

从用户查询中提取：
- **查询类型**：OP名 / OP_CODE / 能力维度 / OP类别
- **目标对象**：具体的 Operation 名称（如 matmul）、OP_CODE（如 OP_A_MUL_B）、约束维度（如 bf16 支持）、功能域（如 cube）
- **关注维度**：Layout、Stride、地址对齐、硬件参数、TileShape、数据类型等

### 2. 知识检索阶段

根据查询类型，按以下路径检索知识库：

**按OP名查询：**
1. 在 `constraints/opcode_mapping.yaml` 中查找 Operation 条目
2. 获取 `category`（功能域）和 `pipeline`（执行链路）
3. 根据 pipeline 中每个 stage 的 `constraint_file` 路径，读取对应的约束 YAML 文件
4. 同时读取 `_common.yaml` 获取公共硬件常量

**按OP_CODE查询：**
1. 根据 OP_CODE 名称，在 `tileop_constraints/` 的分类子目录中定位
2. 可参考 `INDEX.md` 的"各约束文件 OP_CODE 清单"快速定位
3. 读取对应 YAML 文件中该 OP_CODE 的约束规则
4. 同时读取 `_common.yaml` 获取公共定义

**按能力维度查询：**
1. 先读取 `_common.yaml` 获取全局约束（如 `dtype_constraints`、`tag_sets`）
2. 遍历 `tileop_constraints/` 下所有分类目录的 YAML 文件
3. 按目标维度过滤（如筛选支持 bf16 的 OP_CODE、支持 dynamic_unaligned 的 OP_CODE）
4. 汇总输出

**按OP类别查询：**
1. 根据类别名（cube/vector/move/view/misc）进入对应子目录
2. 读取该目录下所有 YAML 文件
3. 汇总输出该类别的 OP_CODE 清单和共性约束

### 3. 约束提取阶段

从检索到的 YAML 文件中提取以下约束维度：

1. **Layout 约束**：支持的存储格式（ND/NZ/FRACTAL_Z/NC1HWC0）
2. **Stride 约束**：步长对齐要求（BLOCK_SIZE 的整数倍等）
3. **地址对齐约束**：起始地址对齐要求（GM/L1/UB/L0A/L0B/L0C 各存储层级）
4. **硬件参数约束**：REPEAT_MAX、MASK_LEN、BLOCK_SIZE 等硬件限制
5. **TileShape 约束**：分块形状要求（如 16 的整数倍等）
6. **数据类型约束**：支持的数据类型（fp16/bf16/fp32/int8 等）
7. **标签**：从 `_common.yaml` 的 `tag_sets` 中关联的标签（如 dynamic_unaligned、binary_op 等）

### 4. 报告生成阶段

按照 `references/report_template.md` 格式输出报告。

**报告必须包含：**
1. 基本信息（OP名、OP_CODE、类别）
2. Pipeline 概览（对于按OP名查询，展示完整执行链路）
3. 约束详情（按阶段列出各维度约束）
4. 标签（关联的 tag_sets 标签）
5. 备注（特殊注意事项）
6. 来源（引用的 YAML 文件路径）

## 输出文件规则

输出直接显示在对话中，不生成文件。

## 注意事项

1. **每个约束必须包含来源引用** -- 引用具体的 YAML 文件路径
2. **如果无法确定，明确说明** -- 不要猜测，说明需要进一步验证
3. **区分公共约束和专属约束** -- `_common.yaml` 中的约束是全局共享的，需要单独说明
4. **注意 OP_CODE 的触发条件** -- 同一 Operation 可能因条件不同使用不同 OP_CODE（如 add 可能用 OP_ADD、OP_ADD_BRC 或 OP_ADDS）
5. **pipeline 是逻辑视图** -- opcode_mapping.yaml 中的 pipeline 是逻辑执行链路，实际代码生成可能融合或省略部分环节

## 知识库文件

### 入口与索引
- `constraints/INDEX.md` -- 知识库总索引，含 OP_CODE 清单和快速查询表
- `constraints/opcode_mapping.yaml` -- Layer 1: Operation API -> OP_CODE 执行链路映射

### 公共定义
- `constraints/tileop_constraints/_common.yaml` -- 硬件常量、对齐规则、数据类型约束、标签集

### 分类约束文件
- `constraints/tileop_constraints/cube/` -- 矩阵计算单元（AIC Core）：conv.yaml、matmul.yaml
- `constraints/tileop_constraints/vector/` -- 向量计算单元（AIV Core）：elementwise.yaml、elementwise_scalar.yaml、reduce.yaml、sort.yaml、bitwise.yaml、broadcast.yaml、cast.yaml、compare.yaml、logical.yaml、gather_scatter.yaml
- `constraints/tileop_constraints/move/` -- 数据搬运：copy_in.yaml、copy_out.yaml、local_move.yaml、gather_load.yaml
- `constraints/tileop_constraints/view/` -- 视图与形状操作：view.yaml、reshape.yaml、expand.yaml、assemble.yaml
- `constraints/tileop_constraints/misc/` -- 杂项操作：special.yaml、index.yaml、duplicate.yaml、register.yaml

### 参考模板
- `references/query_guide.md` -- 查询模式详细指南
- `references/report_template.md` -- 报告输出模板

## 示例用法

### 示例1：按OP名查询

**输入：**
```
/pypto-op-capability-checker matmul 支持 bf16 吗
```

**预期输出：**
1. 在 opcode_mapping.yaml 中查找 matmul 条目
2. 识别类别为 cube，核心 OP_CODE 为 OP_A_MUL_B 等
3. 读取 cube/matmul.yaml 中的 dtype 约束
4. 输出报告：matmul 的各 OP_CODE 对 bf16 的支持情况

### 示例2：按OP_CODE查询

**输入：**
```
/pypto-op-capability-checker OP_ADD 的约束
```

**预期输出：**
1. 在 vector/elementwise.yaml 中查找 OP_ADD 条目
2. 提取 Layout、Stride、地址对齐、硬件参数、TileShape 等约束
3. 关联 _common.yaml 中的公共定义
4. 输出完整约束报告

### 示例3：按能力维度查询

**输入：**
```
/pypto-op-capability-checker 哪些 OP_CODE 不支持 bf16
```

**预期输出：**
1. 遍历所有约束 YAML 文件
2. 筛选 dtype 支持列表中不包含 bf16 的 OP_CODE
3. 汇总输出列表（如 OP_EXP、OP_RSQRT、OP_MOD 等）

### 示例4：按OP类别查询

**输入：**
```
/pypto-op-capability-checker cube 类 OP 有哪些约束
```

**预期输出：**
1. 进入 cube/ 目录
2. 读取 conv.yaml 和 matmul.yaml
3. 汇总 cube 类的共性约束（如 NZ/FRACTAL_Z Layout 要求、M/K/N 维度对齐等）
4. 列出该类别下所有 OP_CODE 及核心约束
