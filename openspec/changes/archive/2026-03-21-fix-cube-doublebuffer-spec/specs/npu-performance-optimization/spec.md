## MODIFIED Requirements

### Requirement: Cube DoubleBuffer L1 Buffer Allocation Diagram

The L1 Buffer diagram in section 3.3 SHALL show all three buffer types required for Cube DoubleBuffer optimization: left matrix, right matrix, and accumulator result.

#### Scenario: L1 Buffer shows complete allocation
- **WHEN** a developer reads section 3.3 L1 Buffer diagram
- **THEN** the diagram SHALL display:
  - Left matrix DoubleBuffer blocks (Block0/Block1)
  - Right matrix DoubleBuffer blocks (Block0/Block1)
  - Accumulator result DoubleBuffer blocks (Block0/Block1)

### Requirement: Cube DoubleBuffer Pipeline Execution Order

The time axis in section 3.3 SHALL reflect the correct execution order as defined in section 3.2's dependency analysis: right matrix loads first, then left matrix loads in parallel with L1→L0B transfer.

#### Scenario: MTE2 loads right matrix before left matrix
- **WHEN** a developer reads the MTE2 time axis in section 3.3
- **THEN** the sequence SHALL show right matrix loading before left matrix for each iteration: `[右0] [左0] [右1] [左1]`

#### Scenario: MTE1 label is explicit about transfer content
- **WHEN** a developer reads the MTE1 time axis in section 3.3
- **THEN** the label SHALL explicitly indicate right matrix transfer: `[右0→L0B] [右1→L0B] ...`
