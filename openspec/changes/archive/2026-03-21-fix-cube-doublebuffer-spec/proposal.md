## Why

The Cube DoubleBuffer optimization section (3.3) in `npu-performance-optimization/spec.md` contains two documentation errors that mislead readers about the actual data flow and buffer allocation:

1. **Timeline inconsistency**: The time axis contradicts the dependency analysis in section 3.2, showing wrong data load order.
2. **Missing buffer diagram**: The L1 Buffer diagram omits the right matrix buffer, creating confusion with section 7.1's capacity calculation.

Accurate documentation is critical for developers implementing NPU kernels with DoubleBuffer optimization.

## What Changes

- **Fix 3.3 L1 Buffer diagram**: Add right matrix DoubleBuffer blocks to match actual data flow (`GM → L1 → L0B`)
- **Fix 3.3 time axis**: Correct MTE2 load order from `[左0][右0][左1][右1]` to `[右0][左0][右1][左1]` (right matrix must load first)
- **Clarify MTE1 label**: Make it explicit that MTE1 is transferring right matrix from L1 to L0B

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `npu-performance-optimization`: Correcting section 3.3 to accurately describe Cube DoubleBuffer pipeline execution order and L1 buffer allocation requirements.

## Impact

- **Documentation only**: No code changes
- **Affected file**: `openspec/specs/npu-performance-optimization/spec.md`
- **Sections affected**: 3.3 (L1 + L0B 多级 DoubleBuffer)
