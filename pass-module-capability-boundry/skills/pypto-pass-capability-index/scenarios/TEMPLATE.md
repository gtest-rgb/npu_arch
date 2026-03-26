# SCXXX: [场景名称]

## 元信息
- 场景ID: SCXXX
- 创建日期: YYYY-MM-DD
- 最后更新: YYYY-MM-DD
- 状态: 草稿/评审中/已验证
- 维护者: [姓名/团队]

## 场景标签

### OPCode标签
[选择适用的标签，参考 `tags/opcode_tags.yaml`]
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
[选择适用的标签，参考 `tags/ir_level_tags.yaml`]
- [ ] TENSOR_GRAPH
- [ ] TILE_GRAPH
- [ ] BLOCK_GRAPH

### 问题类型标签
[选择适用的标签，参考 `tags/problem_type_tags.yaml`]
- [ ] MEMORY_TYPE
- [ ] DATA_LAYOUT
- [ ] SCHEDULING
- [ ] SYNC
- [ ] MEMORY_REUSE
- [ ] GRAPH_OPTIMIZATION
- [ ] DYNAMIC_SHAPE
- [ ] TENSOR_SPLIT

### 特殊场景标签
[选择适用的标签，参考 `tags/special_case_tags.yaml`]
- [ ] CUBE_CASCADE
- [ ] DYNAMIC_SHAPE
- [ ] MULTI_OUTPUT
- [ ] LARGE_TENSOR
- [ ] LOOP_NESTING
- [ ] MIX_SUBGRAPH
- [ ] PADDING

## 场景描述

### 业务背景
[描述这个场景在业务中的背景和需求]

### 典型IR结构
```
[使用ASCII图或代码块描述典型的Operation和Tensor结构]

例如：
MATMUL_1 --> ASSEMBLE --> [大Tensor] --> VIEW --> MATMUL_2
```

### 触发条件
[描述什么情况下会触发这个场景]
1. 条件1
2. 条件2
3. ...

## 涉及Pass

| Pass名称 | 在此场景中的作用 | 约束条件 | 风险等级 | 文档链接 |
|----------|------------------|----------|----------|----------|
| [Pass1] | [作用描述] | [约束] | 高/中/低 | [链接] |
| [Pass2] | [作用描述] | [约束] | 高/中/低 | [链接] |

## 快速检查清单

### 必须满足的条件
- [ ] [条件1]
- [ ] [条件2]

### 建议检查的项目
- [ ] [检查项1]
- [ ] [检查项2]

### 常见遗漏项
- [ ] [容易遗漏的检查项1]
- [ ] [容易遗漏的检查项2]

## 典型Case引用

### 正确案例
- **案例1**：`[文件路径]`
  - 描述：[简要描述这个案例]
- **案例2**：`[文件路径]`
  - 描述：[简要描述这个案例]

### 错误案例
- **案例1**：`[文件路径]`（如果有的话）
  - 描述：[描述错误原因和修复方法]

## 常见问题

| 问题现象 | 可能原因 | 定位方法 | 解决方案 |
|----------|----------|----------|----------|
| [现象1] | [原因1] | [方法1] | [方案1] |
| [现象2] | [原因2] | [方法2] | [方案2] |

## 相关文档

### Pass文档
- [Pass1能力边界]：`.agents/skills/pypto-pass/pypto-pass-module-analyzer/XX_PASS1.md#8-能力边界`
- [Pass2能力边界]：`.agents/skills/pypto-pass/pypto-pass-module-analyzer/XX_PASS2.md#8-能力边界`

### 模块文档
- [Passes模块能力边界]：`docs/passes/PASSES_MODULE_CAPABILITY_BOUNDARY.md`

### 示例代码
- [示例1]：`examples/[路径]`
- [示例2]：`models/[路径]`
