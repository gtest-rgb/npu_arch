## 1. Update L1 Buffer Diagram

- [x] 1.1 Update section 3.3 L1 Buffer diagram header from `L1 Buffer (左矩阵/累加)` to `L1 Buffer (左矩阵/右矩阵/累加)`
- [x] 1.2 Add right matrix DoubleBuffer blocks to the L1 Buffer diagram (between left matrix and accumulator sections)

## 2. Fix Time Axis Execution Order

- [x] 2.1 Fix MTE2 time axis sequence from `[左0] [右0] [左1] [右1]` to `[右0] [左0] [右1] [左1]`
- [x] 2.2 Update MTE1 label from `[0] [1] [0] [1]` to `[右0→L0B] [右1→L0B] [右0→L0B] ...`

## 3. Verification

- [x] 3.1 Verify section 3.3 is now consistent with section 3.2's dependency analysis
- [x] 3.2 Verify L1 Buffer diagram matches section 7.1's capacity formula
