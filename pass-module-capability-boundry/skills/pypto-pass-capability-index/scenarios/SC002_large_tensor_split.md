# SC002: 大Tensor拆分场景

## 元信息
- 场景ID: SC002
- 创建日期: 2026-03-26
- 最后更新: 2026-03-26
- 状态: 已验证
- 维护者: Passes模块负责人

## 场景标签

### OPCode标签
- [ ] MATMUL
- [ ] VIEW
- [ ] ASSEMBLE
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
- [ ] MEMORY_TYPE
- [ ] DATA_LAYOUT
- [ ] SCHEDULING
- [ ] SYNC
- [ ] MEMORY_REUSE
- [ ] GRAPH_OPTIMIZATION
- [ ] DYNAMIC_SHAPE
- [x] TENSOR_SPLIT

### 特殊场景标签
- [ ] CUBE_CASCADE
- [ ] DYNAMIC_SHAPE
- [x] MULTI_OUTPUT
- [x] LARGE_TENSOR
- [ ] LOOP_NESTING
- [ ] MIX_SUBGRAPH
- [ ] PADDING

## 场景描述

### 业务背景
当Tensor过大（超过片上内存容量）或Fanout过多（一个Tensor被多个消费者使用）时，需要对Tensor进行拆分，以便于内存分配和调度。

### 典型IR结构
```
[大Tensor A] --> SplitRawTensor --> [小Tensor 1] --> Consumer 1
                         --> [小Tensor 2] --> Consumer 2
                         --> [小Tensor 3] --> Consumer 3
```

### 触发条件
1. Tensor大小超过L1/UB容量阈值
2. Tensor的Fanout数量超过阈值（通常为8）
3. Reshape操作的输入Tensor过大

## 涉及Pass

| Pass名称 | 在此场景中的作用 | 约束条件 | 风险等级 | 文档链接 |
|----------|------------------|----------|----------|----------|
| SplitRawTensor | 将大Tensor拆分成多个小Tensor | 拆分维度整除 | 中 | [待补充] |
| SplitLargeFanoutTensor | 将高Fanout的Tensor拆分 | Fanout > 阈值 | 中 | [待补充] |
| SplitReshape | 拆分过大的Reshape操作 | 输入输出Shape可拆分 | 低 | [待补充] |
| AssignMemoryType | 为拆分后的Tensor分配内存类型 | - | 低 | [09_ASSIGN_MEMORY_TYPE.md](../../pypto-pass-module-analyzer/09_ASSIGN_MEMORY_TYPE.md) |

## 快速检查清单

### 必须满足的条件
- [x] 拆分后的子Tensor维度是否正确
- [x] 拆分后的子Tensor数量是否合理

### 建议检查的项目
- [ ] 拆分策略是否最优（影响后续内存分配）
- [ ] 拆分后是否引入过多额外操作

### 常见遗漏项
- [ ] 拆分阈值是否正确设置
- [ ] 是否考虑了数据依赖关系

## 常见问题

| 问题现象 | 可能原因 | 定位方法 | 解决方案 |
|----------|----------|----------|----------|
| 拆分后内存反而增加 | 拆分粒度过细 | 检查拆分后的Tensor数量 | 调整拆分阈值 |
| 拆分后性能下降 | 引入过多额外Copy | 检查新增的Operation数量 | 减少拆分或合并消费者 |
| 拆分维度不整除 | 维度选择不当 | 检查Shape维度 | 选择可整除的维度 |

## 相关文档

### Pass文档
- AssignMemoryType能力边界：`.agents/skills/pypto-pass/pypto-pass-module-analyzer/09_ASSIGN_MEMORY_TYPE.md#8-能力边界`

### 模块文档
- Passes模块能力边界：`docs/passes/PASSES_MODULE_CAPABILITY_BOUNDARY.md`
