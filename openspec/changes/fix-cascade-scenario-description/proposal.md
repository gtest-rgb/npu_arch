## Why

当前 `npu-performance-optimization` spec 第4节的级联场景描述存在概念混淆：将 FixPipe 随路处理能力（量化、激活、反量化等）描述为需要独立 Vector 计算阶段，导致读者对 Cube→Vector 级联场景产生误解。

实际情况是：FixPipe 作为 Cube 的随路处理单元，可以在矩阵乘计算过程中同步完成量化、激活等操作，无需单独的 Vector 后处理阶段。只有当需要进行一般 Vector 算子（如 Softmax、LayerNorm、Add 等）时，才需要 Cube→Vector 级联。

## What Changes

- 修正第4节级联场景的概念描述，区分 **FixPipe 随路处理** 与 **独立 Vector 算子**
- 更新 4.3 Cube→Vector 级联的场景描述，明确该模式适用于一般 Vector 算子
- 更新 4.6 典型算子示例，修正或替换错误示例
- 添加 FixPipe 随路处理的说明，将其视为 Cube 计算单元的能力

## Capabilities

### New Capabilities

无

### Modified Capabilities

- `npu-performance-optimization`: 修正级联场景描述，区分 FixPipe 随路处理与独立 Vector 算子

## Impact

- 影响文件：`openspec/specs/npu-performance-optimization/spec.md`
- 影响章节：第4节 Vector + Cube 级联优化（4.1-4.6）
- 不影响代码实现，仅修正文档描述
