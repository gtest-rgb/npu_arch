# SC001: Cube级联场景

## 元信息
- 场景ID: SC001
- 创建日期: 2026-03-26
- 最后更新: 2026-03-26
- 状态: 已验证
- 维护者: Passes模块负责人

## 场景标签

### OPCode标签
- [x] MATMUL
- [x] VIEW
- [x] ASSEMBLE
- [ ] RESHAPE
- [ ] COPY
- [ ] NOP
- [ ] LOOP
- [ ] REDUCE
- [ ] ELEMENTWISE

### IR层级标签
- [ ] TENSOR_GRAPH
- [x] TILE_GRAPH
- [ ] BLOCK_GRAPH

### 问题类型标签
- [x] MEMORY_TYPE
- [x] DATA_LAYOUT
- [ ] SCHEDULING
- [ ] SYNC
- [ ] MEMORY_REUSE
- [ ] GRAPH_OPTIMIZATION
- [ ] DYNAMIC_SHAPE
- [ ] TENSOR_SPLIT

### 特殊场景标签
- [x] CUBE_CASCADE
- [ ] DYNAMIC_SHAPE
- [ ] MULTI_OUTPUT
- [ ] LARGE_TENSOR
- [ ] LOOP_NESTING
- [ ] MIX_SUBGRAPH
- [ ] PADDING

## 场景描述

### 业务背景
Cube级联是LLM推理中的常见模式，特别是在Attention和MoE（Mixture of Experts）结构中。该场景需要将多个MATMUL操作的结果通过ASSEMBLE（小搬大）组合成大的中间Tensor，然后再通过VIEW（大搬小）拆分给后续的MATMUL消费。

这种模式的挑战在于：
1. 内存类型需要在L0C（MATMUL输出）和L1（中间Buffer）之间正确传递
2. ASSEMBLE的offset必须满足对齐要求
3. VIEW/ASSEMBLE的嵌套深度不能过深

### 典型IR结构
```
# Attention场景示例
MATMUL_QK --> VIEW --> ASSEMBLE_QK --> [大Tensor] --> VIEW --> SOFTMAX
MATMUL_AV --> ASSEMBLE_AV --> [大Tensor] --> VIEW --> OUTPUT

# MoE场景示例
MATMUL_GATE --> VIEW --> ASSEMBLE_GATE --> [大Tensor] --> VIEW --> TOPK
MATMUL_EXPERT_* --> ASSEMBLE_EXPERTS --> [大Tensor] --> VIEW --> COMBINE
```

### 触发条件
1. 多个MATMUL操作需要级联执行
2. 需要将多个小Tile组合成大Tile处理（小搬大）
3. 需要将大Tile拆分成小Tile供后续消费（大搬小）
4. 中间Tensor大小超过单个Tile容量

## 涉及Pass

| Pass名称 | 在此场景中的作用 | 约束条件 | 风险等级 | 文档链接 |
|----------|------------------|----------|----------|----------|
| AssignMemoryType | 为MATMUL输入输出分配内存类型，处理ASSEMBLE/VIEW的内存类型传递 | offset 32B对齐；小搬大维度整除 | 中 | [09_ASSIGN_MEMORY_TYPE.md](../../pypto-pass-module-analyzer/09_ASSIGN_MEMORY_TYPE.md) |
| MergeViewAssemble | 合并相邻的VIEW和ASSEMBLE操作 | 嵌套深度≤3 | 低 | [待补充] |
| IntraSubgraphAdapter | 调整数据布局以适应硬件要求 | - | 低 | [待补充] |
| GenerateMoveOp | 根据内存类型插入Move操作 | 内存路径可达 | 中 | [待补充] |

## 快速检查清单

### 必须满足的条件
- [x] ASSEMBLE的offset是否32B对齐
- [x] 小Tile到大Tile的维度是否满足整除关系
- [x] ASSEMBLE输出是否超过UB阈值(35%)
- [x] 内存路径是否可达（L0C→L1）

### 建议检查的项目
- [ ] VIEW/ASSEMBLE/RESHAPE嵌套深度是否超过3层
- [ ] 中间Tensor的生命周期是否正确管理
- [ ] 是否有更优的内存复用方案

### 常见遗漏项
- [ ] 忘记检查offset对齐，导致强制走DDR
- [ ] 忽略ASSEMBLE输出大小，超过UB阈值后降级
- [ ] 嵌套深度过深导致性能下降

## 典型Case引用

### 正确案例
- **案例1**：`examples/03_advanced/advanced_nn/attention/attention.py`
  - 描述：标准的Flash Attention实现，展示了正确的Cube级联模式
- **案例2**：`models/deepseek_v32_exp/`
  - 描述：DeepSeek MoE实现，展示了复杂的Cube级联场景

### 错误案例
- **案例1**：offset未对齐
  - 描述：ASSEMBLE offset不是32B对齐，导致输出被强制走DDR
  - 修复：调整数据布局，确保offset 32B对齐

## 常见问题

| 问题现象 | 可能原因 | 定位方法 | 解决方案 |
|----------|----------|----------|----------|
| ASSEMBLE输出被强制走DDR | offset未32B对齐 | 检查AssembleOpAttribute的offset | 调整数据布局确保对齐 |
| 嵌套深度超过3层警告 | VIEW/ASSEMBLE嵌套过深 | 查看日志警告 | 重构IR结构减少嵌套 |
| 大Tensor内存溢出 | ASSEMBLE输出超过UB阈值 | 检查输出Tensor大小 | 考虑分块处理或使用DDR |
| 内存路径不可达 | L0C→L1路径不在预定义路径中 | 检查Move操作的fromType/toType | 检查硬件支持的内存路径 |
| 小搬大维度不整除 | 大Tile维度不能被小Tile整除 | 检查Tile维度关系 | 调整Tile切分策略 |

## 相关文档

### Pass文档
- AssignMemoryType能力边界：`.agents/skills/pypto-pass/pypto-pass-module-analyzer/09_ASSIGN_MEMORY_TYPE.md#8-能力边界`

### 模块文档
- Passes模块能力边界：`docs/passes/PASSES_MODULE_CAPABILITY_BOUNDARY.md`

### 示例代码
- Attention实现：`examples/03_advanced/advanced_nn/attention/attention.py`
- MoE实现：`models/deepseek_v32_exp/`
