---
name: pypto-pass-capability-checker
description: Passes模块能力边界判断工具。用于判断给定场景在Passes模块中的能力边界，输出可行性判断、约束条件和风险提示。当需要快速判断"这个场景Passes能否正确处理"时使用此技能。
---

# PyPTO Pass Capability Checker Skill

## 概述

本技能用于判断给定场景在Passes模块中的能力边界，输出可行性判断、约束条件和风险提示。

### 使用场景

- 团队内部评审：讨论设计方案时，快速判断可行性
- 对外技术支持：回答用户/开发者关于支持范围的问题

### 触发机制

手动调用：`/pypto-pass-capability-checker <场景描述>`

### 功能概述

1. 场景理解：从描述中提取关键特征
2. 澄清问答：如果场景描述不清晰，追问关键信息
3. 知识检索：匹配场景标签，关联相关Pass
4. 综合判断：输出可行性判断和风险提示
5. 引用依据：每个判断都有文档/代码引用

## 工作流程

按照以下流程执行：

### 1. 场景理解阶段

从用户描述中提取：
- 涉及的OPCode（MATMUL, VIEW, ASSEMBLE, RESHAPE, COPY, NOP, LOOP, REDUCE, ELEMENTWISE等）
- IR层级（TENSOR_GRAPH / TILE_GRAPH / BLOCK_GRAPH）
- 场景特征（特殊场景标签：CUBE_CASCADE, DYNAMIC_SHAPE, MULTI_OUTPUT, LARGE_TENSOR等）
- 问题类型（MEMORY_TYPE, DATA_LAYOUT, SCHEDULING, SYNC, MEMORY_REUSE等）

### 2. 澄清问答阶段（可选）

如果场景描述不够清晰，追问关键信息。参见 `references/clarification_questions.md`。

关键澄清点：
- IR层级确认
- OPCode确认
- 数据规模确认
- Shape相关确认
- 内存相关确认
- 级联相关确认
- 约束相关确认

### 3. 知识检索阶段

1. 根据场景特征，在 `.agents/skills/pypto-pass/pypto-pass-capability-index/` 中匹配场景
   - 先查看 `INDEX.md` 中的场景列表和快速查询表
   - 根据OPCode、问题类型、特殊场景标签查找匹配场景

2. 获取相关Pass列表
   - 从场景索引中获取涉及的Pass
   - 从标签文件中获取相关Pass

3. 读取相关Pass的能力边界章节
   - 读取 `.agents/skills/pypto-pass/pypto-pass-module-analyzer/XX_PASS_NAME.md` 中的 §8 能力边界章节
   - 同时参考 `docs/passes/PASSES_MODULE_CAPABILITY_BOUNDARY.md` 中的模块整体约束

### 4. 综合判断阶段

基于检索结果，综合判断：

**整体可行性判断标准：**
- ✅ **支持**：场景在Passes职责范围内，且所有相关Pass都明确支持
- ⚠️ **有约束**：场景在Passes职责范围内，但需要满足特定约束条件
- ❌ **不支持**：场景不在Passes职责范围内，或相关Pass明确不支持

**判断要素：**
- 相关Pass及各自的约束条件
- 必须满足的约束
- 潜在风险点
- 建议的检查项

### 5. 输出报告阶段

按照 `references/judgment_template.md` 格式输出报告。

**报告必须包含：**
1. 场景理解摘要
2. 明确的判断结论（支持/有约束/不支持）
3. 相关Pass及约束表格
4. 必须满足的约束列表（含引用依据）
5. 潜在风险点列表（含引用依据）
6. 建议检查项
7. 参考文档链接

## 输出文件规则

输出直接显示在对话中，不生成文件。

## 注意事项

1. **每个判断必须包含引用依据** - 引用具体的文档路径和章节
2. **如果无法确定，明确说明** - 不要猜测，说明需要进一步验证
3. **输出格式必须统一** - 便于阅读和追溯
4. **优先匹配现有场景** - 先在场景索引中查找，再分析单个Pass
5. **跨Pass交互要考虑** - 多个Pass之间的依赖和互斥关系

## 知识库文件

### 模块整体能力边界
- `docs/passes/PASSES_MODULE_CAPABILITY_BOUNDARY.md`

### 场景索引
- `.agents/skills/pypto-pass/pypto-pass-capability-index/INDEX.md`
- `.agents/skills/pypto-pass/pypto-pass-capability-index/tags/` - 标签定义
- `.agents/skills/pypto-pass/pypto-pass-capability-index/scenarios/` - 场景索引

### Pass文档
- `.agents/skills/pypto-pass/pypto-pass-module-analyzer/*.md`

### 参考模板
- `references/judgment_template.md` - 判断报告模板
- `references/clarification_questions.md` - 澄清问题模板

## 示例用法

### 示例1：Cube级联场景

**输入：**
```
/pypto-pass-capability-checker 用户有一个场景：多个MATMUL通过ASSEMBLE组合成大Tensor，然后再通过VIEW拆分给后续MATMUL使用
```

**预期输出：**
1. 匹配场景：SC001 Cube级联
2. 识别OPCode：MATMUL, ASSEMBLE, VIEW
3. 识别IR层级：TILE_GRAPH
4. 检索相关Pass：AssignMemoryType, MergeViewAssemble
5. 输出判断报告（包含约束：offset 32B对齐、嵌套深度≤3等）

### 示例2：模糊描述

**输入：**
```
/pypto-pass-capability-checker 用户想做一个矩阵乘法
```

**预期输出：**
1. 触发澄清问答
2. 追问：IR层级、数据规模、是否有级联、内存要求等
3. 获取更多信息后继续流程

### 示例3：不支持场景

**输入：**
```
/pypto-pass-capability-checker 用户想在Passes中做代码生成的指令调度
```

**预期输出：**
1. 识别为CodeGen职责
2. 判断为不支持
3. 说明应由CodeGen层处理
