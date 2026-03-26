# SC003: OOO调度场景

## 元信息
- 场景ID: SC003
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
- [ ] TILE_GRAPH
- [x] BLOCK_GRAPH

### 问题类型标签
- [ ] MEMORY_TYPE
- [ ] DATA_LAYOUT
- [x] SCHEDULING
- [x] SYNC
- [x] MEMORY_REUSE
- [ ] GRAPH_OPTIMIZATION
- [ ] DYNAMIC_SHAPE
- [ ] TENSOR_SPLIT

### 特殊场景标签
- [ ] CUBE_CASCADE
- [ ] DYNAMIC_SHAPE
- [ ] MULTI_OUTPUT
- [ ] LARGE_TENSOR
- [ ] LOOP_NESTING
- [x] MIX_SUBGRAPH
- [ ] PADDING

## 场景描述

### 业务背景
OOO（Out-of-Order）调度是一种乱序执行策略，允许Block Graph中的操作根据数据依赖关系和资源可用性进行调度，而不是严格按照拓扑顺序执行。这种策略可以提高硬件利用率和整体性能。

### 典型IR结构
```
Block Graph:
  Block_A (MATMUL) --> Block_B (ElementWise) --> Block_C (MATMUL)
  Block_D (MATMUL) --> Block_E (ElementWise) --> Block_F (MATMUL)

OOO Scheduling后:
  时间片1: Block_A, Block_D (并行执行)
  时间片2: Block_B, Block_E (并行执行)
  时间片3: Block_C, Block_F (并行执行)
```

### 触发条件
1. 使用PVC2_OOO编译策略
2. Block Graph中有可并行执行的操作
3. 硬件支持OOO调度模式

## 涉及Pass

| Pass名称 | 在此场景中的作用 | 约束条件 | 风险等级 | 文档链接 |
|----------|------------------|----------|----------|----------|
| OoOSchedule | 进行乱序调度，依赖分析正确 | 中 | [待补充] |
| InsertSync | 插入同步点 | 依赖关系正确 | 中 | [待补充] |
| GlobalMemoryReuse | 全局内存复用 | 生命周期不重叠 | 低 | [待补充] |
| TuneSyncForVF | 优化同步点 | - | 低 | [待补充] |
| TuneTileOpSeqForVF | 优化Tile操作序列 | - | 低 | [待补充] |

## 快速检查清单

### 必须满足的条件
- [x] 依赖分析是否正确
- [x] 同步点是否正确插入
- [x] 内存复用是否安全

### 建议检查的项目
- [ ] 调度效率是否最优
- [ ] 同步点是否过多（影响性能）
- [ ] 并行度是否充分利用

### 常见遗漏项
- [ ] 跨子图依赖是否处理
- [ ] 动态Shape对调度的影响
- [ ] 硬件资源限制

## 常见问题

| 问题现象 | 可能原因 | 定位方法 | 解决方案 |
|----------|----------|----------|----------|
| 性能不如预期 | 同步点过多 | 检查InsertSync输出 | 调整同步策略 |
| 数据竞争 | 同步点缺失 | 检查依赖关系 | 添加必要的同步点 |
| 内存冲突 | 内存复用错误 | 检查GlobalMemoryReuse输出 | 调整复用策略 |
| 调度效率低 | 依赖分析不完整 | 检查OoOSchedule输入 | 检查PreCheck是否通过 |

## 相关文档

### Pass文档
- AssignMemoryType能力边界：`.agents/skills/pypto-pass/pypto-pass-module-analyzer/09_ASSIGN_MEMORY_TYPE.md#8-能力边界`

### 模块文档
- Passes模块能力边界：`docs/passes/PASSES_MODULE_CAPABILITY_BOUNDARY.md`
