## Context

This is a documentation accuracy fix for `openspec/specs/npu-performance-optimization/spec.md` section 3.3. The section describes the L1 + L0B multi-level DoubleBuffer optimization for Cube (matrix multiplication) operations.

Two issues were identified:
1. **Timeline order error**: The time axis shows `[左0][右0][左1][右1]` but section 3.2's dependency analysis shows right matrix must load first before L1→L0B can begin.
2. **Incomplete buffer diagram**: L1 Buffer diagram only shows left matrix and accumulator, missing right matrix buffer that section 7.1's capacity formula explicitly includes.

## Goals / Non-Goals

**Goals:**
- Correct the L1 Buffer diagram to show all three buffer types: left matrix, right matrix, and accumulator
- Fix the time axis to reflect correct execution order: right matrix loads first, then left matrix loads in parallel with L1→L0B
- Clarify MTE1 label to indicate it transfers right matrix

**Non-Goals:**
- Changing any technical content beyond the documented errors
- Modifying section 3.2, 3.4, 3.5, or any other sections
- Changing the conceptual explanation of DoubleBuffer optimization

## Decisions

### Decision 1: L1 Buffer Diagram Structure

**Choice**: Add right matrix DoubleBuffer blocks between left matrix and accumulator sections.

**Rationale**: The data flow in 3.1 and 3.2 clearly shows right matrix path as `GM → L1 → L0B`. The right matrix must reside in L1 before being transferred to L0B. Section 7.1's capacity formula confirms this: `右矩阵 = stepN × stepKb × 2(DB) × sizeof(dtype)`.

**Layout**:
```
L1 Buffer (左矩阵/右矩阵/累加)
├── Block0/Block1 (左矩阵) - Ping-Pong
├── Block0/Block1 (右矩阵) - Ping-Pong  ← ADDED
└── Block0/Block1 (累加结果) - Ping-Pong
```

### Decision 2: Time Axis Correction

**Choice**: Change MTE2 sequence from `[左0][右0][左1][右1]` to `[右0][左0][右1][左1]`

**Rationale**: Section 3.2's dependency diagram shows:
```
Load右(N) ──完成──▶ L1→L0B(N) ──┬──完成──▶ MMAD(N)
            │                   │
            └──▶ Load左(N) ─────┘
```
This means: right matrix must complete loading → then L1→L0B and Load左 can proceed in parallel. Therefore, right matrix loads first.

### Decision 3: MTE1 Label Clarification

**Choice**: Change `[0] [1] [0] [1]` to `[右0→L0B] [右1→L0B] ...`

**Rationale**: The original label is ambiguous. Making it explicit clarifies that MTE1 transfers the right matrix from L1 to L0B buffer.

## Risks / Trade-offs

| Risk | Mitigation |
|------|------------|
| Readers may have already internalized the incorrect diagram | The fix is clearly documented; no migration needed since this is reference documentation |
| Diagram becomes more complex with additional buffer | The added complexity is necessary for accuracy; matches section 7.1's formula |
