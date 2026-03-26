# Passes模块能力边界

> 版本：1.0
> 日期：2026-03-26
> 维护者：Passes模块负责人

## 1. 模块定位

### 1.1 在编译流水线中的位置

Passes模块位于PyPTO编译流水线的核心层，负责将用户定义的计算图逐步变换为可执行的硬件指令。

```
Python API (用户代码)
    ↓
Tensor Graph (张量级IR)
    ↓
┌───────────────────────────────────────┐
│           PASSES MODULE                │
│  ┌─────────────────────────────────┐  │
│  │ Tensor Graph Passes             │  │  ← LoopUnroll, ExpandFunction,
│  │                                  │  │    RemoveRedundantReshape, AutoCast,
│  │                                  │  │    InferMemoryConflict, ...
│  └─────────────────────────────────┘  │
│                ↓                       │
│  ┌─────────────────────────────────┐  │
│  │ Tile Graph Passes               │  │  ← AssignMemoryType, MergeViewAssemble,
│  │                                  │  │    SplitReshape, GraphPartition,
│  │                                  │  │    IntraSubgraphAdapter, ...
│  └─────────────────────────────────┘  │
│                ↓                       │
│  ┌─────────────────────────────────┐  │
│  │ Block Graph Passes              │  │  ← GlobalMemoryReuse, OoOSchedule,
│  │                                  │  │    InsertSync, CodegenPreproc, ...
│  └─────────────────────────────────┘  │
└───────────────────────────────────────┘
    ↓
Block Graph / Execution Graph (可执行IR)
    ↓
CodeGen (硬件指令生成)
```

**编译策略：** Passes模块支持多种编译策略（如 `PVC2_OOO`、`FunctionUnroll`、`ExecuteGraph`），每种策略定义了不同的Pass执行序列。

### 1.2 核心职责

**负责：**
- IR变换与优化：在不同IR层级之间进行图变换和优化
- 内存分配：为Tensor分配合适的内存类型（L0A/L0B/L0C/L1/UB/DDR）
- 调度优化：指令级并行调度、内存复用、同步插入
- 图结构优化：消除冗余操作、合并视图/组装操作、拆分大Tensor
- 约束传播：推断和传播Shape、MemoryType等约束信息

**不负责：**
- 用户代码解析：由Python API层处理
- 硬件指令生成：由CodeGen层处理
- 运行时执行：由Runtime层处理
- 数值计算：只进行图变换，不执行实际计算

## 2. 输入边界

### 2.1 接收的IR类型

Passes模块接收并处理三种IR类型的Function对象：

**Tensor Graph（张量图）**
- **格式要求**：由`LogicalTensor`和`Operation`构成的有向无环图（DAG）
- **节点类型**：
  - Tensor节点：表示逻辑张量，包含Shape、DataType、Layout等属性
  - Operation节点：表示计算操作，通过Opcode区分操作类型（如OP_RESHAPE、OP_VIEW、OP_MATMUL等）
- **边类型**：Tensor作为Operation的输入输出，形成Producer-Consumer关系
- **图结构**：Function包含Incast（输入张量列表）和Outcast（输出张量列表）
- **适用Pass**：RemoveRedundantReshape、AutoCast、InferMemoryConflict、LoopUnroll、ExpandFunction等

**Tile Graph（分块图）**
- **格式要求**：在Tensor Graph基础上增加了内存类型（MemoryType）和Tile相关信息
- **扩展属性**：
  - MemoryType：MEM_L0A、MEM_L0B、MEM_L0C、MEM_L1、MEM_UB、MEM_DEVICE_DDR等
  - Move操作：OP_L1_TO_L0A、OP_L1_TO_L0B等内存搬移操作
  - View/Assemble操作：OP_VIEW、OP_ASSEMBLE，用于描述内存视图变换
- **适用Pass**：AssignMemoryType、MergeViewAssemble、SplitReshape、GraphPartition、IntraSubgraphAdapter等

**Block Graph（块图）**
- **格式要求**：包含子图（SubGraph）结构的执行图，每个子图可独立调度
- **扩展结构**：
  - SubGraph：可独立调度的计算单元
  - Buffer信息：内存复用相关的缓冲区管理
  - Schedule信息：操作执行顺序和依赖关系
- **适用Pass**：GlobalMemoryReuse、OoOSchedule、InsertSync、CodegenPreproc、CopyOutResolve等

### 2.2 输入约束条件

**通用约束（适用于所有IR类型）**
1. **图结构完整性**
   - Function必须包含非空的Incast和Outcast
   - 所有Operation必须非空（CheckValidOp）
   - 所有Operation的输入输出Tensor必须非空（CheckOpIOValid）
   - 图中不能存在环（CheckGraphLoop）

2. **张量有效性**
   - 所有Tensor必须有有效的Producer和Consumer（CheckConsumerProducer）
   - 局部定义的Tensor必须有Producer（CheckLocalTensor）
   - Tensor的Shape、DataType、Layout等属性必须合法

3. **操作连接性**
   - Operation之间的连接必须通过Tensor建立Producer-Consumer关系
   - 不能存在悬空的操作（既无输入也无输出）

**Tensor Graph特定约束**
- 动态Shape约束：OP_VIEW的fromDynOffset_和toDynValidShape_必须非空
- 动态Shape约束：OP_ASSEMBLE的toDynOffset_必须非空
- Reshape操作：输入输出Shape必须满足reshape语义

**Tile Graph特定约束**
- 内存类型可达性：Move操作的源MemoryType到目标MemoryType必须在预定义路径中
  - 合法路径示例：MEM_L0C → MEM_L1、MEM_L1 → MEM_L0A、MEM_DEVICE_DDR → MEM_L1等
- View/Assemble操作：toType属性必须合法（如MEM_BT、MEM_FIX_QUANT_PRE、MEM_L0A、MEM_L0B）
- A_MUL_B操作：输入Producer必须是OP_L1_TO_L0A/L0B、OP_VIEW或OP_VEC_DUP

**Block Graph特定约束**
- 子图拓扑结构必须正确（CheckSubGraphTopo）
- 调度前的依赖关系必须正确

### 2.3 输入验证机制

Passes模块通过**PreCheck**机制在每个Pass执行前验证输入是否满足约束条件。

**PreCheck调用时机**
```cpp
Status Pass::Run(Function &function, const std::string &strategy,
                 const std::string &identifier, size_t runtimeIdx) {
    // 1. 执行PreCheck（输入验证）
    if (PreCheck(function) != SUCCESS) {
        ALOG_ERROR_F("PreCheck failed for pass %s.", identifier.c_str());
        return FAILED;
    }
    // 2. 执行Pass核心逻辑
    if (RunOnFunction(function) != SUCCESS) {
        return FAILED;
    }
    // 3. 执行PostCheck（输出验证）
    if (PostCheck(function) != SUCCESS) {
        ALOG_ERROR_F("PostCheck failed for pass %s.", identifier.c_str());
        return FAILED;
    }
    return SUCCESS;
}
```

**PreCheck验证内容**

每个Pass可以重写`PreCheck`方法实现特定的输入验证：

1. **公共验证（PublicCheck）**
   - 检查Operation是否有效（非空）
   - 检查Operation的输入输出Tensor是否有效
   - 检查Function的Incast/Outcast完整性
   - 检查图是否存在环
   - 检查局部Tensor有效性

2. **Pass特定验证示例**

   **AssignMemoryType::PreCheck**
   - 检查A_MUL_B操作的输入Producer合法性
   - 检查View/Assemble/Reshape嵌套深度（不超过3层）
   - 防止内存分配不合理导致性能下降

   **RemoveRedundantReshape::PreCheck**
   - 检查Reshape操作必须有Consumer
   - 防止删除无Consumer的Reshape导致图断开

   **OoOSchedule::PreCheck**
   - 检查Tensor信息完整性（PreCheckTensorInfo）
   - 检查Operation信息完整性（PreCheckOpInfo）
   - 保存原始Function状态用于PostCheck对比

   **SubgraphToFunction::PreCheck**
   - 检查子图拓扑结构（CheckSubGraphTopo）
   - 检查NOP操作的合法性
   - 检查子图边界的完整性

**PreCheck失败处理**
- PreCheck失败时，Pass立即返回FAILED，终止执行
- 错误信息通过APASS_LOG_ERROR_F记录，包含Operation Magic ID和堆栈信息
- 支持通过`GetFormatBacktrace`输出完整的调用栈，便于调试

## 3. 输出边界

### 3.1 产出的IR类型

Passes模块的输出IR类型取决于执行的编译策略和Pass序列，通常遵循以下变换路径：

**Tensor Graph → Tile Graph变换**
- **触发Pass**：AssignMemoryType及其后续Pass
- **变换特征**：
  - 为所有Tensor分配MemoryType属性（从MEM_UNKNOWN变为具体内存类型）
  - 插入Move操作（如OP_L1_TO_L0A）用于内存搬移
  - 可能插入View/Assemble操作用于内存视图变换
- **输出格式**：包含内存类型信息的Tile Graph

**Tile Graph → Block Graph变换**
- **触发Pass**：GraphPartition、SubgraphToFunction
- **变换特征**：
  - 将Tile图分割为多个SubGraph（子图）
  - 每个子图可独立调度和执行
  - 生成子图间的依赖关系
- **输出格式**：包含SubGraph结构的Block Graph

**Block Graph → Execution Graph变换**
- **触发Pass**：OoOSchedule、GlobalMemoryReuse、InsertSync、CodegenPreproc
- **变换特征**：
  - 完成操作调度（确定执行顺序）
  - 完成内存复用（减少内存占用）
  - 插入同步操作（保证正确性）
  - 预处理CodeGen所需信息
- **输出格式**：可执行的Execution Graph，可直接送入CodeGen生成硬件指令

**渐进式变换**
- Passes模块支持渐进式编译，可通过配置在某阶段终止
- 示例：设置`COMPILE_STAGE=CS_TENSOR_GRAPH`可在ExpandFunction后终止
- 终止点：ExpandFunction（Tensor Graph阶段）、SubgraphToFunction（Tile Graph阶段）

### 3.2 输出保证

Passes模块通过**PostCheck**机制确保每个Pass的输出满足正确性和不变性约束。

**PostCheck调用时机**
```cpp
Status Pass::Run(Function &function, ...) {
    // Pass执行后立即进行PostCheck
    if (PostCheck(function) != SUCCESS) {
        ALOG_ERROR_F("PostCheck failed for pass %s.", identifier.c_str());
        return FAILED;
    }
    return SUCCESS;
}
```

**不变性保证**

1. **图结构不变性**
   - PostCheck后图中仍不能有环（CheckGraphLoop）
   - 所有Operation仍必须有效（CheckValidOp）
   - Function的Incast/Outcast必须保持完整（CheckCompleteness）

2. **Pass特定保证示例**

   **AssignMemoryType::PostCheck**
   - **CheckTensorNotMemUnknown**：所有Tensor的MemoryType必须已分配（不能是MEM_UNKNOWN）
   - **CheckMoveOpReachable**：所有Move操作的源→目标MemoryType必须在预定义路径中
   - **保证**：内存分配完备且可达

   **RemoveRedundantReshape::PostCheck**
   - **CheckForConsecutiveReshape**：不能存在连续的Reshape操作
   - **保证**：消除了冗余的Reshape，图结构得到简化

   **OoOSchedule::PostCheck**
   - **PostCheckOpMagic**：检查Operation Magic ID唯一性
   - **PostCheckTensorMagic**：检查Tensor Magic ID唯一性
   - **PostCheckNewOpConnection**：检查新插入Operation的连接正确性
   - **PostCheckLocalTensor**：检查局部Tensor的Producer关系
   - **PostCheckGlobalTensor**：检查全局Tensor的完整性
   - **PostCheckDynValidShape**：检查动态Shape的有效性
   - **保证**：调度后的图结构完整、连接正确、ID唯一

   **SubgraphToFunction::PostCheck**
   - **CheckInAndOutGraphMatch**：检查子图输入输出边的匹配性
   - **ColorOutGraphCheck**：检查子图颜色（用于区分不同子图）的正确性
   - **VerifySingleOpTopology**：验证单操作子图的拓扑结构
   - **CheckReadyStateConsistency**：检查子图就绪状态的一致性
   - **保证**：子图结构正确、边界清晰、依赖关系正确

3. **正确性保证**
   - **语义等价性**：Pass变换不改变计算语义（除了优化Pass消除冗余操作）
   - **类型安全**：所有Tensor的DataType、Shape、MemoryType在变换后保持一致
   - **连接完整性**：Producer-Consumer关系在变换后保持完整

**PostCheck失败处理**
- PostCheck失败时，Pass返回FAILED，编译流水线终止
- 错误信息记录详细的失败原因和位置（Operation/Tensor Magic ID）
- 支持通过Dump机制保存Pass前后的图结构，便于调试分析

### 3.3 输出限制

**已知限制**

1. **IR层级限制**
   - Passes模块不生成硬件指令，最终输出仍是IR（Execution Graph）
   - 硬件指令生成由CodeGen模块负责
   - Passes输出的IR必须满足CodeGen的输入要求

2. **优化限制**
   - **局部优化**：大多数Pass只进行局部优化，不考虑全局最优
   - **启发式算法**：某些Pass（如OoOSchedule、GlobalMemoryReuse）使用启发式算法，可能不是最优解
   - **性能不可预测**：Pass执行时间可能与图大小、硬件配置强相关

3. **内存分配限制**
   - **MemoryType决策**：AssignMemoryType基于规则和启发式，可能不是最优分配
   - **内存复用**：GlobalMemoryReuse可能无法找到最优复用方案
   - **Buffer冲突**：某些场景下可能出现Buffer冲突，需要通过InferMemoryConflict提前检测

4. **图结构限制**
   - **SubGraph划分**：GraphPartition的子图划分可能不适合所有场景
   - **View/Assemble嵌套**：过深的View/Assemble嵌套（>3层）会触发警告，可能影响性能
   - **大Tensor拆分**：SplitLargeFanoutTensor、SplitReshape等Pass可能无法处理极端情况

5. **动态Shape限制**
   - **动态Shape推断**：InferDynShape只能推断部分动态Shape，复杂场景可能失败
   - **动态属性传播**：某些动态属性（如fromDynOffset、toDynValidShape）可能在Pass链中传播失败

6. **硬件相关限制**
   - **架构依赖**：某些Pass只支持特定NPU架构（通过supportedArches_配置）
   - **资源约束**：Pass不考虑实际硬件资源限制（如L1/L0大小），由Runtime层处理
   - **同步点插入**：InsertSync基于保守策略，可能插入过多同步点影响性能

## 4. 不处理的场景

### 4.1 职责范围外的问题

以下问题**不在Passes模块职责范围内**，应通过其他机制处理：

| 问题类型 | 说明 | 应由谁处理 |
|----------|------|------------|
| 用户代码语义错误 | 如维度不匹配、类型不兼容等 | Python API层（编译前检查） |
| 数值精度问题 | 如浮点精度损失、溢出等 | 算法层/Kernel层 |
| 并发正确性 | 如多流并发的数据竞争 | Runtime层（流管理） |
| 硬件故障 | 如NPU设备异常 | Runtime层（设备管理） |
| 资源超限 | 如总内存超出设备容量 | Runtime层（资源管理） |
| Kernel实现 | 如具体算子的计算逻辑 | CodeGen + Kernel库 |

### 4.2 应由上游处理的场景

以下场景应由**Python API层**处理后再传入Passes：

| 场景 | 上游职责 | 期望的输入状态 |
|------|----------|----------------|
| 动态Shape缺失 | Python层应提供动态Shape的计算方式 | 动态Shape属性已设置 |
| 非法图结构 | Python层应验证图的基本合法性 | 图结构完整、无孤立节点 |
| 类型推断 | Python层应完成基本类型推断 | DataType已确定 |
| 用户属性传递 | Python层应设置必要的Op属性 | Op属性完整 |

### 4.3 应由下游处理的场景

以下场景应由**CodeGen层**或**Runtime层**处理：

| 场景 | 下游职责 | Passes的输出状态 |
|------|----------|------------------|
| 指令生成 | CodeGen负责生成硬件指令 | 输出Block Graph IR |
| 内存分配实现 | Runtime负责实际内存分配 | 输出包含MemoryType的IR |
| 执行调度 | Runtime负责实际执行顺序 | 输出包含调度信息的IR |
| 硬件同步 | Runtime/CodeGen负责实际同步 | 输出包含同步点的IR |
| 错误处理 | Runtime负责运行时错误处理 | Passes不处理运行时错误 |

## 5. 上下游接口

### 5.1 与Python API的接口

**职责分界：**

| 层级 | 职责 | 接口 |
|------|------|------|
| Python API | 用户接口、语义表达、图构建 | Function/Operation/Tensor对象 |
| Passes | 图优化、内存分配、调度 | 对Function进行变换 |

**输入约定：**
- Python API构建的Function对象应满足：
  - Incast/Outcast非空
  - 所有Operation有有效的Opcode
  - 所有Tensor有有效的Shape和DataType
  - 图结构无环

**输出保证：**
- Passes处理后的Function：
  - 保持语义等价（除非是消除冗余操作）
  - 内存类型已分配
  - 调度信息已确定

### 5.2 与CodeGen的接口

**职责分界：**

| 层级 | 职责 | 接口 |
|------|------|------|
| Passes | 图优化、内存分配、调度规划 | Block Graph IR |
| CodeGen | 指令生成、二进制输出 | PTO指令/二进制 |

**输入约定（CodeGen对Passes的期望）：**
- Block Graph结构完整
- 所有Tensor有确定的MemoryType
- 调度顺序已确定
- 同步点已插入

**输出保证（Passes对CodeGen的承诺）：**
- 满足CodeGen输入要求
- 内存路径可达
- 依赖关系正确

### 5.3 职责分界示例

**示例1：Tile Shape设置**

```
Python API层：
  - 职责：用户提供Tile Shape信息
  - 代码：pypto.set_cube_tile_shapes(...)

Passes层：
  - 职责：根据Tile Shape进行内存分配和优化
  - Pass：AssignMemoryType

CodeGen层：
  - 职责：根据Tile Shape生成具体指令
  - 不负责：Tile Shape的选择和验证
```

**示例2：内存类型冲突处理**

```
Python API层：
  - 不处理：内存类型冲突
  - 只负责：表达用户意图

Passes层：
  - 职责：检测并处理内存类型冲突
  - Pass：InferMemoryConflict, AssignMemoryType
  - 行为：自动插入Convert操作解决冲突

CodeGen层：
  - 不处理：内存类型冲突（假设已解决）
  - 只负责：根据内存类型生成Move指令
```

**示例3：动态Shape处理**

```
Python API层：
  - 职责：标记动态维度，提供Shape计算逻辑
  - 代码：pypto.mark_dynamic(...)

Passes层：
  - 职责：推断动态Shape，传播动态属性
  - Pass：InferDynShape, DynAttrToStatic
  - 限制：只能处理可推断的动态Shape

Runtime层：
  - 职责：根据实际Shape执行计算
  - 不处理：Shape推断
```

## 6. 整体约束

### 6.1 硬件相关约束

**内存层级约束：**

| 内存层级 | 容量（参考） | 访问延迟 | 用途 |
|----------|-------------|----------|------|
| L0A/L0B | ~1MB | 最低 | Cube输入缓冲 |
| L0C | ~1MB | 最低 | Cube输出缓冲 |
| L1 | ~8MB | 低 | 中间数据存储 |
| UB | ~8MB | 低 | 通用缓冲区 |
| DDR | ~32GB | 高 | 大容量存储 |

**对齐约束：**

| 约束项 | 对齐要求 | 违反后果 |
|--------|----------|----------|
| ASSEMBLE offset | 32B对齐 | 强制走DDR |
| L1 buffer首地址 | 64B对齐 | 访问效率下降 |
| Cube操作输入 | 特定对齐要求 | 功能异常 |

---

## 7. 已知问题与未来工作

### 7.1 已知限制
1. **内存类型推断不完整性**：某些复杂场景下，MEM_UNKNOWN可能无法自动推断
2. **动态Shape支持有限**：InferDynShape只能处理部分动态Shape场景
3. **OOO调度保守策略**：InsertSync可能插入过多同步点

### 7.2 未来改进方向
1. **智能内存分配**：基于图特征自动选择最优内存分配策略
2. **精确同步点插入**：基于数据依赖分析优化同步点位置
3. **动态Shape完整支持**：扩展InferDynShape的处理能力

### 7.3 版本历史

| 版本 | 日期 | 修改内容 |
|------|------|----------|
| 1.0 | 2026-03-26 | 初始版本，| 1.1 | 2026-03-26 | 补充输入输出边界、| 1.2 | 2026-03-26 | 补充不处理场景和上下游接口 |
| 1.3 | 2026-03-26 | 补充整体约束章节 |

**架构约束：**

| 架构 | 特殊约束 | 影响的Pass |
|------|----------|------------|
| PVC2 | 支持OOO调度 | OoOSchedule |
| PVC2 | 特定内存路径 | AssignMemoryType |
| 910B | 不支持某些优化 | 通过supportedArches过滤 |

### 6.2 性能相关约束

**优化策略选择：**

| 场景 | 推荐策略 | 原因 |
|------|----------|------|
| 大模型推理 | PVC2_OOO | 支持乱序调度，性能更好 |
| 小模型推理 | ExecuteGraph | 简单场景，编译更快 |
| 调试阶段 | FunctionUnroll | 展开循环便于调试 |

**性能相关阈值：**

| 阈值项 | 默认值 | 说明 |
|--------|--------|------|
| UB内存使用上限 | 35% | 超过会降级到DDR |
| L1内存使用上限 | 50% | 超过会降级到DDR |
| View/Assemble嵌套深度 | 3层 | 超过会警告 |
| 单个子图操作数上限 | 架构相关 | 影响子图划分 |

**性能优化建议：**

1. **减少内存搬运**
   - 合理设置Tile Shape，减少ASSEMBLE/VIEW
   - 确保offset对齐，避免强制DDR

2. **优化调度**
   - 使用OOO策略提高并行度
   - 减少不必要的同步点

3. **控制图复杂度**
   - 避免过深的VIEW/ASSEMBLE嵌套
   - 合理拆分大Tensor
