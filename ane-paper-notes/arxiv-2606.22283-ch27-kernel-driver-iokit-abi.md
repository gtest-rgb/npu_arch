# 第27章《Kernel driver and IOKit ABI》笔记

> 论文：*Apple Neural Engine: Architecture, Programming, and Performance*
> 作者：Spencer H. Bryngelson（Georgia Institute of Technology）
> 提交：2026-06-21，302 页，CC-BY 0
> 本章原文：pp. 171–179
>
> 本章是 ch6 execution runtime 的**下层**——e5rt_* 在用户态，它跨进程跨内核到底要调谁、过哪些 gate、到哪个 MMIO 寄存器。回答：**driver 是谁、selector ABI 长什么样、entitlement 怎么卡、提交一路降到 doorbell 的四层路径**。

---

## 1. driver 栈：3 个 kext 协作

| Kext 角色 | Bundle | 主类 | Provider match |
|---|---|---|---|
| interface + 硬件 driver | `AppleH11ANEInterface` | `ANEHWDevice`（注册名 `H11ANEIn`） | `RTBuddyService`，role = `ANE` |
| per-die 抽象层 | `AppleT8132ANEHAL` | `AppleT8132ANEHAL` | `IOResources` |
| 多引擎 arbiter | `AppleANELoadBalancer` | `ANEDriver`（`H1xANELoadBalancer`） | `IOResources`, `IOKit` |

M1 世代三个 kext 全部 build `9.511.3` 版本锁死。

**职责切分**：
- interface driver：注册设备、对外 vend user client、按铃到固件
- HAL：提供 clock / power / topology 常数
- load balancer：own 程序到引擎的 residency map，多引擎部件时仲裁

引擎以 **Apple RTKit coprocessor endpoint** 挂上来——interface driver 是一个 RTOS mailbox client 的 host 侧。

---

## 2. user client：3 种，按 type 分发

`ANEHWDevice::newUserClient(task*, void*, uint type, IOUserClient**)` 按 type 发出三种 client：

```c
/* 由 newUserClient 按 type 发出： */
H11ANEInUserClient       /* control client：程序生命周期 + 推理热路径（17 个 selector） */
H11ANEInDirectPathClient /* direct path：enqueue + memory-map + session-hint（9 个 selector） */
ANEClientHints           /* scheduling-hint client（setClientHint） */
```

- **两个 functional client 持有独立连接和独立 selector 空间**，都从 0 开始
- 同一个 selector 整数（如 `2`）在两个 client 上**指代不同的方法**
- 每个 client 把 connection 存在 object offset `+0x40`
- 用户态 `IOServiceOpen` 返回 `0xe00002c5` 表示打开设备成功，connection 对象位于 device handle 的 `+0x40`

### 2.1 两个前端

| 前端 | 用途 |
|---|---|
| `ANEServicesDevice` | 推理 |
| `ANEHWDevice` | 管理 |

---

## 3. selector ABI：扁平 + 精确 size tuple

### 3.1 总览

| Client | Selector 数 | dispatch-array 地址 | dispatch 机制 |
|---|:-:|---|---|
| `H11ANEInUserClient` | 17 (0–16) | `_sANEDriverClientMethods` @ +0xc08 | `IOUserClient2022::dispatchExternalMethod` |
| `H11ANEInDirectPathClient` | 9 (0–8) | `_sANEDriverDirectPathClientMethods` @ +0xeb0 | 同上 |

### 3.2 dispatch-array 元素布局（40 字节 stride）

```c
struct IOExternalMethodDispatch2022 {
    void    *function;                  /* +0x00 PAC 签名的 handler 指针 */
    uint32_t checkScalarInputCount;     /* +0x08 精确 scalar 输入个数 */
    uint32_t checkStructureInputSize;   /* +0x0c 精确 struct 输入字节 */
    uint32_t checkScalarOutputCount;    /* +0x10 精确 scalar 输出个数 */
    uint32_t checkStructureOutputSize;  /* +0x14 精确 struct 输出字节 */
    uint8_t  reserved[0x10];            /* +0x18 保留（debug-WP group flag） */
};
```

**关键约束**：
- 没有任何 selector 使用哨兵 `0xffffffff`（"不检查"）
- 每个 selector 锁死一个精确的 size tuple
- dispatcher 对任何其他 size 返回 `kIOReturnBadArgument` (`0xe00002c2`)
- **没有"按 size 重载"**：一个 selector 索引 = 一个 record = 一个固定 size tuple

### 3.3 IOExternalMethodArguments 字段偏移（所有 selector shim 共用）

```
+0x08  asyncWakePort         /* mach completion port（仅 async selector） */
+0x10  scalarInput[]         /* 也承载 0x20 字节 asyncReference 块 */
+0x20  scalarInputCount
+0x30  structureInput        /* 指向 typed argument struct */
+0x38  structureInputSize
+0x48  scalarOutput[]
+0x50  scalarOutputCount
+0x58  structureOutput
+0x60  structureOutputSize
```

### 3.4 三个返回码

| 码 | 含义 |
|---|---|
| `0xe00002c2` | `kIOReturnBadArgument`——size 或 null-pointer check 失败 |
| `0xe00002c7` | `kIOReturnUnsupported`——禁用或 stub 路径（ch8 entitlement gate 也用它） |
| `0xe00002c5` | client 已关闭或正在关闭 |

---

## 4. Control-client 17 个 selector（§27.3）

| Sel | Handler | Scalar in | Struct in | Scalar out | Struct out |
|:-:|---|:-:|:-:|:-:|:-:|
| 0 | `ANE_DeviceOpen` | 0 | 104 | 0 | 104 |
| 1 | `ANE_DeviceClose` | 0 | 0 | 0 | 0 |
| **2** | **`ANE_ProgramSendRequest`** | **1** | **2376** | **0** | **40** |
| 3 | `ANE_ProgramCreate` | 0 | 32 | 0 | 0 |
| 4 | `ANE_ProgramPrepare` | 0 | 56 | 0 | 56 |
| 5 | `ANE_ProgramUnprepare` | 0 | 56 | 0 | 0 |
| 6 | `ANE_ProgramDestroy` | 0 | 16 | 0 | 0 |
| 7 | `ANE_GetStatus` | 0 | 0 | 0 | 32 |
| 8 | `ANE_ProgramCreateInstance` | 0 | 32 | 0 | 0 |
| 9 | `ANE_ProgramChainingPrepare` | 0 | 16 | 0 | 24 |
| 10 | `ANE_GetVersion` | 0 | 0 | 1 | 0 |
| 11 | `ANE_RegisterDebugWorkProcessor` | 0 | 24 | 0 | 0 |
| 12 | `ANE_UnregisterDebugWorkProcessor` | 0 | 0 | 0 | 0 |
| 13 | `ANE_GetDebugWorkProcessorItem` | 2 | 0 | 0 | 0 |
| 14 | `ANE_CompleteDebugWorkProcessorItem` | 2 | 0 | 0 | 0 |
| 15 | `ANE_ReleaseDebugWorkProcessorBuffers` | 0 | 0 | 0 | 0 |
| 16 | `ANE_LoadFirmware` | 3 | 0 | 0 | 0 |

**几个关键点**：

- **selector 0**：open handshake。104 字节 `ANEDeviceInfo` 既输入又输出，回填 session token（`+0x00`）、ANE 版本（`+0x48`/`+0x50`）、引擎数（`+0x50`）、CPU subtype（`+0x60`）
- **selector 2**：**唯一**对 hardware doorbell 的 user-space 入口
- **selector 16**：当前 build 上 inactive，shim 无条件返回 `kIOReturnUnsupported`，真实 firmware load 在 driver start 时内部跑

### 4.1 selector 2 的 2376 字节请求结构

```
+0x000  program / instance token (u64)   /* program-create 时铸造 */
+0x008  sequence number                   /* 0, 1, 2, ... */
+0x010  priority / QoS pair               /* 实测 (5, 21) */
+0x01c  io category                       /* 实测 2 */
+0x020  surface identifier array          /* 输入/输出/中间 buffer 的 surface id */
```

40 字节输出：`+0x00` sequence + result, `+0x08` 回显 token, `+0x20` 状态标志。

---

## 5. Direct-path 9 个 selector（§27.4）

| Sel | Handler | Scalar in | Struct in | Scalar out | Struct out |
|:-:|---|:-:|:-:|:-:|:-:|
| 0 | `ANE_DeviceOpen` | 0 | 104 | 0 | 104 |
| 1 | `ANE_DeviceClose` | 0 | 0 | 0 | 0 |
| **2** | **`ANE_ProgramSendRequest`** | 1 | 2376 | 0 | 40 |
| 3 | `ANE_ProgramOutputSetEnqueue` | 0 | 40 | 0 | 0 |
| 4 | `ANE_ProgramInputsReady` | 0 | 3104 | 0 | 0 |
| 5 | `ANE_MemoryMapRequest` | 1 | 2080 | 1 | 0 |
| 6 | `ANE_MemoryUnMapRequest` | 0 | 2080 | 0 | 0 |
| 7 | `ANE_SessionHintRequest` | 0 | 16 | 0 | 24 |
| 8 | `ANE_ProgramChainingSetActiveProcedure` | 0 | 32 | 0 | 0 |

- **selector 0/1/2** 复用 control client 的 handler
- **3 / 4**：resident submission 模型的预登记 + 触发——output buffer set 入队、输入就绪信号、同样走 selector 2 的 doorbell 路径
- **5 / 6**：设备 IOMMU map/unmap，2080 字节描述 host buffer；成功后把 engine-visible device address 写到唯一 scalar 输出槽
- **7 / 8**：session-hint + 跨步骤 procedure 切换

---

## 6. selector 2 的四层下降路径（§27.6）

```
H11ANEInUserClient::externalMethod(sel=2, args)
   │
   ▼  dispatchExternalMethod（校验 2376-in / 40-out）
   │
   ▼  ANE_ProgramSendRequest(client, ref, args)    /* arg shim */
   │
   ▼  ANEClientDevice::programSendRequest(ANEProgramRequestArgs*, ...)
   │
   ▼  ANEDriver::ANE_ProgramSendRequest(...)        /* gated */
   │
   ▼  ANEHWDevice::doorBellRing(db)
   │
   ▼  ANERegisterControl::write32(reg, 1 << idx)   /* MMIO mailbox */
```

每层职责：
- **shim**：再校验 arg size、解出 typed argument struct
- **client 对象方法**：建 memory descriptor、retain shared-event fence
- **gated driver 方法**：跑在 command-gate workloop 上
- **doorbell write**：从 request 读 doorbell index（必须 < 32），算 `1 << index`，写进 engine register aperture——这是触发固件的 mailbox 信号

### 6.1 engine → host 完成信号

反向路径同样是 windowed store：
1. 固件清除 AArch64 sysreg `S3_3_C15_C8_0` 的 bit 39 → 打开 posted-write 窗口
2. 固件把 doorbell 值 store 进 host aperture
3. barrier
4. 固件读 uncorrectable-cache-error bit（bit 1）和 transaction-reject bit（bit 7）确认已提交
5. 设置 bit 39 关窗
6. 全程禁中断，防止嵌套 handler 污染 status 读

完成以 mach wake port callback 形式返回，**不是** shared-memory 轮询。

---

## 7. entitlement 与 broker 模型（§27.7）

### 7.1 两个内核 entitlement（client 打开时检查）

```c
/* H11ANEInUserClient::init 经 copyClientEntitlement 检查： */
"com.apple.ane.iokit-user-access"        /* 硬 device-open gate */
"com.apple.ane.allow-dataChaining-access" /* resident data-chaining gate */
```

检查**仅在 client 构造时跑一次**——是个 client 对象上的 boolean，不是每次 selector 重新校验。

### 7.2 谁能直接打开 device？

**全系统正好两个二进制**持有 `com.apple.ane.iokit-user-access`：
- 系统 broker daemon
- 它的 per-user 兄弟进程

**没有任何 application 进程能直接打开 device**。其他所有消费者都通过 broker 跨进程调用，用 broker 自己的私有 entitlement 家族证明身份。

### 7.3 broker 私有 entitlement 家族

| Entitlement | Capability | 持有者数 |
|---|---|:-:|
| `com.apple.ane.iokit-user-access` | 硬内核 gate：打开 user client | 2 |
| `com.apple.ane.allow-dataChaining-access` | resident data-chaining on direct-path | 内核检 |
| `com.apple.aned.private.allow` | baseline：经 broker compile/load/instantiate | 18 |
| `com.apple.aned.private.ANEAccess.allow` | inference-client access-grant 变体 | 14 |
| `com.apple.aned.private.adapterWeight.allow` | 把 adapter 权重 stream 到共享 resident base model | 5 |
| `com.apple.aned.private.processModelShare.allow` | 跨进程共享一个 resident model | 4 |
| `com.apple.aned.private.secondaryANECompilerServiceAccess.allow` | 大模型的长时编译服务 | 1 |
| `com.apple.aned.private.aggressivePowerSaving` | aggressive-power-saving 执行模式 | 仅 gate helper |
| `com.apple.aned.private.modelPurgeInAllPartitions` | 跨所有 cache partition 清理模型 | 仅 gate helper |
| `com.apple.security.temporary-exception.iokit-user-client-class` | 直接开 direct-path user client 自投递 | **27** |
| `com.apple.security.ts.ane-client` | trust-cache blessed-client slot（延迟敏感消费者） | 5 |

### 7.4 broker 模型细节

- **broker 是个 listener**——每连接 + 每方法都做 entitlement 检查
- 三档分类：restricted tier / unrestricted tier / per-user tier
- **QoS 贯穿**：每个 compile / load / instantiate 方法都穿一个 QoS 参数
- **restricted tier** 才放行：adapter-weight / model-share / aggressive-power-saving
- **per-user tier** 服务 per-user broker

### 7.5 adapter-weight 路径（无重编译换权重）

- base model 加载一次
- 每个 adapter 是一个**新 instance**，绑定到具名 base-model identifier
- 只持自己的 per-adapter weight 文件
- 通过 `create-instance-with-weights` 方法，命名 base-model id + weight-file count
- **residency 和 power 是 per-instance 显式参数**：
  - `enable-power-saving` flag
  - `opt-out-of-model-memory-unwiring`（hot client 保留权重常驻）
  - `queue-depth`（broker 在争用时下调）
  - 更激进的 power-saving 变体（restricted tier 才放）

### 7.6 residency 与 code-signing identity 绑死

- 第二个 client **只有 team-id + CDHash 匹配 owner** 才能附加到已 resident 的程序
- 共享 resident model 或 KV cache **不会跨租户泄漏**
- 单物理引擎时，跨 client 仲裁 = 单 gated request queue 上的时分复用，按 per-stream QoS 偏置

---

## 8. 6 个 kernel-driver entitlement（开 gate 之外）

| Entitlement | Capability |
|---|---|
| `com.apple.ane.realtime-priority-client` | real-time-priority client grant |
| `com.apple.ane.allow-system-reserved-priorities` | 系统 reserved 调度优先级 |
| `com.apple.ane.memory` | memory-access grant |
| `com.apple.ane.allow-data` | data-access grant |
| `com.apple.private.ane.allow-set-client-hints` | 设 per-client hints |
| `com.apple.private.ane.allow-share-coalition-hints` | 跨 coalition 共享 hints |

---

## 9. register / power / firmware / exclave 方法（§27.5）

direct-path client **除了 9 个 kernel selector，还对外暴露更广的方法面**——但这些不是独立 selector 索引，而是路由经过 9 个之一或单独入口：

| 方法 | 角色 |
|---|---|
| `ANE_PowerOn` / `PowerOff` / `IsPowered` | 电源域控制 |
| `ANE_LoadFirmware` / `ForgetFirmware` | 固件 image 生命周期 |
| `ANE_SendCommand` | 原始固件命令注入 |
| `ANE_SetPowerManagement` / `SetDynamicPowerGating` / `SetPowerGatingHysterTime` | 电源策略 |
| `ANE_SetThrottlingPercentage` | 热限流 |
| `ANE_SetDARTCacheTTL` / `FlushInactiveDARTMappings` / `UnmapDartBuffers` | 地址翻译控制 |
| `ANE_ReadANERegister` / `WriteANERegister` | **裸 MMIO 寄存器读写** |
| `ANE_FWSharedEventDoorbellRing` | 按固件 shared-event doorbell |
| `ANE_AddPersistentClient` / `RemovePersistentClient` | 让设备常驻 |
| `ANE_MPMMemoryMapRequest` / `MPMMemoryUnmap` | 多进程托管内存 |
| `ANE_ExclaveCycle` / `Load` / `Evaluate` / `Unload` | 安全世界 load/evaluate |
| `ANE_ExclaveReadPropertyValue` / `WritePropertyValue` | 安全世界 property 访问 |
| `ANE_GetClientsInfo` / `ShowSharedMemoryAllocations` / `ShowModelMemoryStatus` | 诊断 |

**裸 register R/W + 命令注入 + exclave 方法**有第二道 access check：client open 时探一个 `privileged-virtual-machine-access` property，**独立于 device-open entitlement**。

---

## 10. 实测 device properties（M1，§27.8）

| Property | Value | 含义 |
|---|---|---|
| architecture | `h13g` | 微架构 family，驱动 per-family codegen |
| version | 96 (`0x60`) | major hardware version |
| minor version | 17 | minor revision |
| board type | 96 | board + SoC type id |
| board subtype | 0 | board sub-variant |
| **number of cores** | **16** | 拓扑意义上的 compute cores（**不是** MAC 阵列宽度） |
| number of engines | 1 | 引擎单元数（M1 = 1，load balancer 直通） |
| CPU subtype | 4 | program-ABI gate |
| internal build | No | release build，门控 debug 表面 |

> "number of cores = 16" 是拓扑计数，**不是吞吐相关的 MAC 阵列宽度**——浮点速率要从 cost-model anchor 测得，不能从 core count 推。静止时注册表显示设备、load-balancer 实例、standing hints client 存在，0 个 control/direct-path client——证实 brokered、lazy-open 模型。

---

## 11. 三条最值得记住的判断

1. **selector ABI 是"扁平 + 精确 size tuple"**——17 + 9 共 26 个 selector，每个锁死一个 (scalarIn, structIn, scalarOut, structOut) tuple，没有 size 重载、没有"不检查"哨兵。dispatcher 对任何偏离返回 `0xe00002c2`。

2. **selector 2 是 user-space 到 hardware doorbell 的唯一入口**——2376 字节请求 + 40 字节回执，经四层下降（externalMethod → ANE_ProgramSendRequest → ANEclientDevice::programSendRequest → ANEDriver → ANEHWDevice::doorBellRing → MMIO write）。其他 selector 都是 lifecycle / map / hint，**不直接按铃**。

3. **entitlement 把"谁能直接开 device"压到全系统 2 个二进制**——`com.apple.ane.iokit-user-access` 只 broker daemon + 它的 per-user 兄弟持有。其他所有应用必须经 broker，用 `com.apple.aned.private.*` 家族证明身份。broker 是 listener，每连接每方法都查。

---

## 12. 与其他章节的勾连

- **→ 第6章（Dispatching without Core ML）**：本章是 ch6 `e5rt_*` 用户态 API 的下层——e5rt 跨进程跨内核下降到本章的 selector + doorbell
- **→ 第8章（Entitlement boundary）**：ch8 的 entitlement gate 用 `0xe00002c7`，本章列出**完整 entitlement 家族**（kernel gate + broker private 家族），把 ch8 的笼统叙述实例化
- **→ 第28章（DART）**：本章 selector 5/6 的 `ANE_MemoryMapRequest` 走的是第28章的 DART IOMMU
- **→ 第29章（Firmware）**：selector 16 `ANE_LoadFirmware` 在当前 build 是 stub，真实 firmware load 在 driver start 时内部跑（第29章详述）
- **→ 第30章（Host-to-firmware command protocol）**：本章 doorbell write 是第30章命令协议的入口
- **→ 第32章（Security and isolation）**：residency 与 code-signing identity 绑死（team-id + CDHash）是第32章 isolation 模型的支点
- **→ 第7章（Weights and compression）**：adapter-weight 路径（§7.5）是 ch7 weight patching / LoRA 工作的 broker 侧实现

---

*本文档基于 arXiv:2606.22283v1 (21 Jun 2026) 第27章 (pp. 171–179) 撰写。论文为逆向工程参考文档，所描述的私有接口未公开、未受支持、跨版本脆弱，仅供研究、测量和设备端实验使用。Core ML 仍是唯一受支持的发行路径。*
