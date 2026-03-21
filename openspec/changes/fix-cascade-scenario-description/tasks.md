## 1. 添加 FixPipe 随路处理说明

- [x] 1.1 在 4.1 级联场景概览后添加新小节 "4.2 FixPipe 随路处理能力"
- [x] 1.2 说明 FixPipe 可随路处理的操作类型（量化、激活、Bias、格式转换）
- [x] 1.3 添加对比表格，区分 FixPipe 随路处理与需要独立 Vector 算子的操作

- [x] 2.1 修正 4.3 Cube → Vector 级联的场景描述，明确该模式适用于独立 Vector 算子（Softmax、LayerNorm 等）
- [x] 2.2 修正 4.4 双向级联的场景描述，强调中间的 Vector 阶段是独立算子而非 FixPipe 随路处理
- [x] 2.3 更新级联模式对比表格，增加 "FixPipe 随路" 作为补充说明
- [x] 3.1 修正示例2 (MatMul → GELU)：说明可通过 FixPipe 随路处理或 Cube→Vector 级联
- [x] 3.2 修正示例3 (Quantize → MatMul → Dequantize)：改为描述 FixPipe 随路处理，无需双向级联
- [x] 3.3 添加或修正示例：MatMul → Softmax 作为真正的 Cube→Vector 级联场景
- [x] 3.4 更新算子示例总结表格
- [x] 4.1 通读修改后的第4节，确保概念清晰无歧义
- [x] 4.2 确认 FixPipe 随路处理与独立 Vector 算子的区分在所有相关章节保持一致

## 2. 删除 Bank冲突相关内容

- [x] 2.1 删除 4.6 节中的 "Bank 冲突考虑" 小节
- [x] 2.2 删除第 9 节 "Unified Buffer Bank 冲突优化"
- [x] 2.3 更新第 10 节标题为 "Scalar 缓存优化"
- [x] 2.4 更新第 10.2 性能收益表格（已无 Bank 冲突相关内容）
- [x] 2.5 更新第 11.2 优化技术汇总表格，删除 Bank 冲突优化行
- [x] 2.6 更新第 11.4 注意事项，删除 Bank 冲突相关说明
- [x] 3.1 删除 4.6 节中的 "Bank 冲突考虑" 小节
