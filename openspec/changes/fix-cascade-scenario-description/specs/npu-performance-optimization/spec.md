## MODIFIED Requirements

### Requirement: Vector + Cube 级联场景分类

级联场景 SHALL 按以下规则分类：

1. **Vector → Cube 级联**: 前置 Vector 算子（如 LayerNorm）完成后，数据通过 MTE3 (UB → L1) 进入 Cube 进行矩阵乘
2. **Cube → Vector 级联**: Cube 矩阵乘完成后，数据通过 FixPipe (L1 → UB) 进入 Vector 进行**独立 Vector 算子**计算
3. **双向级联**: Vector → Cube → Vector 的完整流水线

**关键区分**:
- FixPipe 随路处理（量化、激活、反量化等）视为 Cube 计算单元的能力，不属于 Cube → Vector 级联
- 只有需要**独立 Vector 算子**（如 Softmax、LayerNorm、Add 等）时，才需要 Cube → Vector 级联

#### Scenario: FixPipe 随路处理不触发级联

- **WHEN** Cube 矩阵乘结果需要进行量化、ReLU 激活或反量化
- **THEN** 这些操作 SHALL 通过 FixPipe 随路完成，无需进入独立 Vector 计算阶段

#### Scenario: 独立 Vector 算子触发级联

- **WHEN** Cube 矩阵乘结果需要进行 Softmax、LayerNorm 或一般向量计算
- **THEN** 数据 SHALL 通过 FixPipe (L1 → UB) 进入 Vector 进行独立计算

### Requirement: 级联模式典型算子示例

级联模式的典型算子示例 SHALL 准确反映 FixPipe 随路处理与独立 Vector 算子的区别：

| 算子组合 | 级联模式 | 说明 |
|---------|---------|------|
| LayerNorm → MatMul | Vector→Cube | 前置 LayerNorm + 矩阵乘 |
| MatMul + FixPipe | 无级联 | 矩阵乘 + 随路量化/激活 |
| MatMul → Softmax | Cube→Vector | 矩阵乘 + 独立 Softmax |
| QK^T → Softmax → AV | 多阶段双向 | Attention 完整流水线 |

#### Scenario: 量化推理不使用双向级联

- **WHEN** 执行量化推理 (INT8 矩阵乘)
- **THEN** 量化和反量化 SHALL 通过 FixPipe 随路完成，SHALL NOT 描述为 Vector → Cube → Vector 三级级联

#### Scenario: Attention 使用双向级联

- **WHEN** 执行 Self-Attention 计算
- **THEN** QK^T → Softmax → AV SHALL 描述为 Cube → Vector → Cube 级联，因为 Softmax 是独立 Vector 算子
