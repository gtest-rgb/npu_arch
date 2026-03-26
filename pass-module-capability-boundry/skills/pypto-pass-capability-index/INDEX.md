# Passes场景索引

> 用途：快速定位场景相关Pass和约束条件
> 更新日期: 2026-03-26

## 使用方法

1. 根据场景特征，在 `tags/` 目录查找对应标签
2. 根据标签组合，在 `scenarios/` 目录查找匹配场景
3. 查看场景文档中的涉及Pass和检查清单

## 标签体系

| 维度 | 文件 | 说明 |
|------|------|------|
| OPCode | `tags/opcode_tags.yaml` | 涉及的操作类型 |
| IR层级 | `tags/ir_level_tags.yaml` | Pass作用的IR层级 |
| 问题类型 | `tags/problem_type_tags.yaml` | 可能的问题分类 |
| 特殊场景 | `tags/special_case_tags.yaml` | 需要特别关注的场景 |

## 场景列表

| 场景ID | 场景名称 | 标签 | 文档链接 |
|--------|----------|------|----------|
| SC001 | Cube级联 | MATMUL, ASSEMBLE, VIEW | [scenarios/SC001_matmul_cube_cascade.md](scenarios/SC001_matmul_cube_cascade.md) |
| SC002 | 大Tensor拆分 | TENSOR_SPLIT, MULTI_OUTPUT | [scenarios/SC002_large_tensor_split.md](scenarios/SC002_large_tensor_split.md) |
| SC003 | OOO调度 | SCHEDULING, SYNC, MEMORY_REUSE | [scenarios/SC003_ooo_scheduling.md](scenarios/SC003_ooo_scheduling.md) |

## 快速查询

### 按OPCode查询
- MATMUL: SC001
- ASSEMBLE: SC001,- SC002
- VIEW: SC001

- COPY: SC003

### 按问题类型查询
- 内存类型: SC001
- 数据布局: SC001
- 调度: SC003
- 同步: SC003
- 内存复用: SC003

### 按特殊场景查询
- Cube级联: SC001
- 大Tensor: SC002
- 多输出: SC002
