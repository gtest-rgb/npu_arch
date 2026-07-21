# 直连路径专题：编译与加载的端到端流程

> 论文：*Apple Neural Engine: Architecture, Programming, and Performance*
> 作者：Spencer H. Bryngelson（Georgia Institute of Technology）
> 提交：2026-06-21，302 页，CC-BY 4.0
> 跨章整合：ch5 / ch6 / ch8 / ch26 / ch27
>
> 本文回答一个问题：**直连路径的"编译 + 加载"流程在四个进程里怎么走，每个进程的模块是什么，模块之间的接口是什么，对外暴露给用户的 API 都做些什么。**

---

## 0. 整体地图

直连路径跨四个进程边界：

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  ① 用户进程（user process）                                       │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  application code    ── 用户自己写的                       │  │
│  │         │                                                   │  │
│  │         ▼                                                   │  │
│  │  e5rt_* C dispatch layer（Espresso framework）              │  │
│  │  ★ 对外暴露给用户的接口 ★                                    │  │
│  │         │                                                   │  │
│  │         ▼                                                   │  │
│  │  IOKit user client connection                              │  │
│  │  （H11ANEInUserClient / H11ANEInDirectPathClient）          │  │
│  └─────────────┬──────────────────────────────────────────────┘  │
│                │                                                 │
└────────────────┼─────────────────────────────────────────────────┘
                 │  ★ 接口 1：XPC/Mach IPC + broker entitlement ★
                 │     （com.apple.aned.private.*）
                 ▼
┌──────────────────────────────────────────────────────────────────┐
│  ② 系统 broker daemon（aned）                                     │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  broker listener（per-connection + per-method entitlement）│  │
│  │         │                                                   │  │
│  │         ▼                                                   │  │
│  │  ANECompiler（out-of-process compiler）                    │  │
│  │         │                                                   │  │
│  │         ▼                                                   │  │
│  │  code-signing path（trustcache + corecrypto）              │  │
│  │         │                                                   │  │
│  │         ▼                                                   │  │
│  │  IOKit service client（自己也是 IOKit 客户端）              │  │
│  └─────────────┬──────────────────────────────────────────────┘  │
│                │                                                 │
└────────────────┼─────────────────────────────────────────────────┘
                 │  ★ 接口 2：IOKit selector ABI ★
                 │     （17 control + 9 direct-path selector）
                 ▼
┌──────────────────────────────────────────────────────────────────┐
│  ③ 内核 driver（3 个 kext 协作）                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  AppleH11ANEInterface（interface + 硬件 driver）           │  │
│  │  ├ H11ANEInUserClient       （17 selector）                │  │
│  │  ├ H11ANEInDirectPathClient （9 selector）                 │  │
│  │  └ ANEClientHints                                         │  │
│  │         │                                                   │  │
│  │  AppleT8132ANEHAL（per-die 抽象层）                         │  │
│  │  AppleANELoadBalancer（多引擎 arbiter）                     │  │
│  │         │                                                   │  │
│  │         ▼                                                   │  │
│  │  ANEHWDevice::doorBellRing → MMIO write                    │  │
│  └─────────────┬──────────────────────────────────────────────┘  │
│                │                                                 │
└────────────────┼─────────────────────────────────────────────────┘
                 │  ★ 接口 3：MMIO mailbox + shared-event ring ★
                 ▼
┌──────────────────────────────────────────────────────────────────┐
│  ④ ANE 引擎（silicon + RTKit coprocessor 固件）                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  mailbox controller                                        │  │
│  │  program loader（解析 magic 0xbeefface 程序镜像）            │  │
│  │  compute cores（MAC 阵列）                                  │  │
│  │  DART IOMMU                                                │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1. ① 用户进程的模块清单

| 模块 | 是什么 | 出处 |
|---|---|---|
| **application code** | 用户写的客户端代码（构图、调用 e5rt API、读结果） | ch6 §6.1 |
| **e5rt_* C dispatch layer** | Espresso framework 导出的 C API，共 292 entry point（≈275 可调，≈52 走正常 dispatch） | ch6 §6.1–§6.2 |
| **IOKit user client connection** | 用户进程持有的 IOKit 连接对象，位于 device handle 的 `+0x40` | ch27 §27.1 |

### 1.1 用户进程 ↔ daemon 的接口

| 接口 | 形式 | 方向 |
|---|---|---|
| e5rt 高层 API → 私有 XPC/Mach 调用 | C API 内部跨进程 | 用户 → daemon |
| 返回值（编译产物引用 / signed program handle / surface id） | 跨进程返回 | daemon → 用户 |
| broker 私有 entitlement 家族 | 每连接 + 每方法检查 | 用户自证身份 |

> **注意**：用户进程**不直接调** IOKit。`com.apple.ane.iokit-user-access` 全系统只有 2 个二进制持有（broker daemon + 它的 per-user 兄弟）。用户进程必须经 broker 跨进程调用。

---

## 2. ② 系统 broker daemon 的模块清单

| 模块 | 是什么 | 出处 |
|---|---|---|
| **broker listener** | aned 的入口，per-connection + per-method 检查 entitlement，把客户端分到 restricted / unrestricted / per-user 三档 | ch27 §27.7 |
| **ANECompiler** | out-of-process 编译器，接收 IR（.espresso.net / .mil / .mlpackage），lowering + 验证 + 权重布局 + 写盘到 content-addressed cache | ch6 §6.1、ch22 |
| **code-signing path** | 用 daemon 自身的 trustcache 信任根 + corecrypto 给编译产物签名；信任链：daemon Mach-O 在 trustcache 白名单 → 它生成的临时文件继承信任 | ch8 §7.1、ch25 阶段 7 |
| **IOKit service client** | daemon 自己也是 IOKit 客户端，开 `H11ANEInUserClient` 提交程序到 kernel | ch27 §27.1 |

### 2.1 daemon ↔ kernel 的接口

**接口 2：IOKit selector ABI**（daemon 作为唯一持有 `com.apple.ane.iokit-user-access` 的客户端调）

| Selector | Handler | 用途 | 阶段 |
|:-:|---|---|---|
| 0 | `ANE_DeviceOpen` | 104 字节 `ANEDeviceInfo` 握手，回填 session token | 启动一次 |
| 2 | `ANE_ProgramSendRequest` | **2376 in / 40 out**——提交程序 + 触发 doorbell | dispatch |
| 3 | `ANE_ProgramCreate` | 32 字节——把 daemon 签好的程序 image 注册到 kernel | 加载 |
| 4 | `ANE_ProgramPrepare` | 56 in / 56 out——准备程序运行所需的资源 | 加载 |
| 5 | `ANE_ProgramUnprepare` | 释放 prepare 的资源 | 卸载 |
| 6 | `ANE_ProgramDestroy` | 16 字节——销毁程序实例 | 卸载 |
| 7 | `ANE_GetStatus` | 32 字节状态读 | 诊断 |
| 8 | `ANE_ProgramCreateInstance` | 32 字节——从 base model 创建 adapter instance（adapter-weight 路径） | 加载（变体） |
| 9 | `ANE_ProgramChainingPrepare` | 16 in / 24 out——准备跨步骤 procedure 链 | 高级加载 |
| 16 | `ANE_LoadFirmware` | **当前 build 是 stub**，无条件 `kIOReturnUnsupported` | — |

完整 17 selector 见 [ch27 笔记 §4](./arxiv-2606.22283-ch27-kernel-driver-iokit-abi.md)。

---

## 3. ③ 内核 driver 的模块清单

### 3.1 3 个 kext

| Kext | 职责 |
|---|---|
| `AppleH11ANEInterface` | 注册设备、vend user client、按 doorbell 到固件 |
| `AppleT8132ANEHAL` | 提供 clock / power / topology 常数 |
| `AppleANELoadBalancer` | own 程序→引擎 residency map，多引擎部件仲裁 |

### 3.2 内核 driver ↔ engine 的接口

**接口 3：MMIO mailbox + shared-event ring**

```
ANEDriver::ANE_ProgramSendRequest(...)
   │
   ▼  command-gate workloop
ANEHWDevice::doorBellRing(db)
   │
   ▼  ANERegisterControl::write32(reg, 1 << idx)
MMIO mailbox write   ← 触发固件的硬连线信号
```

- doorbell index 从 request 里读，必须 < 32
- mask = `1 << index`，写到 engine register aperture
- 完成信号反向：固件清 AArch64 sysreg `S3_3_C15_C8_0` bit 39 → posted-write 窗口开 → 把 doorbell store 进 host aperture → barrier → 查 bit 1（uncorrectable-cache-error）和 bit 7（transaction-reject）确认提交 → 设 bit 39 关窗
- 全程禁中断；完成以 mach wake port callback 形式回到用户态

---

## 4. ④ ANE 引擎的模块清单

| 模块 | 是什么 |
|---|---|
| **mailbox controller** | 接收 doorbell 信号，驱动 datapath |
| **program loader** | 解析 magic `0xbeefface` 的 program image（含 register-write text + weight + constants + scratch section） |
| **DART IOMMU** | 16 KB 页粒度、3.5 GiB aperture 的设备 IOMMU（详见 ch28） |
| **compute cores** | M1 上 16 个（拓扑计数，非 MAC 阵列宽度） |
| **RTKit coprocessor 固件** | 引擎本身是 Apple RTKit coprocessor endpoint，跑实时 OS |

---

## 5. ★ 对外暴露给用户的接口（e5rt_* 全表）

**唯一对外暴露给用户的接口面就是 `e5rt_*`**，从 Espresso framework 导出。下表列出五组 family，每个 API 的用途都给出。

### 5.1 compile 阶段（family: `e5rt_e5_compiler_*`）

| API | 用途 |
|---|---|
| `e5rt_e5_compiler_create_with_config(&compiler, config)` | 用一个 config 创建 compiler handle |
| `e5rt_e5_compiler_compile(compiler, model_path, options, &library)` | 把 `.espresso.net` / `.mil` / `.mlpackage` 编译为 program library；内部驱动 daemon 的 out-of-process compiler |
| `e5rt_e5_compiler_is_new_compile_required(...)` | 查 cache 是否需要重编（content-addressed cache hit/miss 判断） |

**编译选项**（string-keyed dict of `std::any`）：

| 键 | 类型 | 用途 |
|---|---|---|
| `forceRecompilation` | `bool` | 强制重编，忽略 cache |
| `segmenter` | `std::string` | 选 segmenter |
| `computeDeviceTypesAllowed` | `std::vector<ComputeDeviceType>` | 设备 mask，`0x4` = ANE |

### 5.2 load 阶段（三个 family 共同协作）

| API | family | 用途 |
|---|---|---|
| `e5rt_program_library_retain_program_function(library, fn_name, &function)` | `e5rt_program_library_*` | retain library 暴露的具名函数 |
| `e5rt_program_library_load_for_execution(...)` | `e5rt_program_library_*` | 把 library 加载进可执行态 |
| `e5rt_precompiled_compute_op_create_options_create_with_program_function(&op_opts, function)` | `e5rt_precompiled_compute_op_*` | 用 function 构造一个 op 的 options |
| `e5rt_execution_stream_operation_create_precompiled_compute_operation_with_options(&op, op_opts)` | `e5rt_execution_stream_*` | 把 options 实例化为 stream operation |

> **load 实际做的事**：runtime 打开 cache 里的 program library，第一次加载付出一次性成本产出"loadable hardware form"——这是从 parametric descriptor 展开成显式 register-write program 的过程（ch2 §2.1 Table 2.2，三阶段：bundle → program image → firmware container）。

### 5.3 bind 阶段

| API | family | 用途 |
|---|---|---|
| `e5rt_buffer_object_alloc(&buf, nbytes, type)` | `e5rt_buffer_object_*` | 分配一个 nbytes 的 buffer object |
| `e5rt_buffer_object_get_data_ptr(buf, &ptr)` | `e5rt_buffer_object_*` | 拿 buffer 的 CPU 可写指针（host 填输入 / 读输出） |
| `e5rt_execution_stream_operation_retain_input_port(op, "x", &in_port)` | `e5rt_execution_stream_*` | retain op 的具名输入端口 |
| `e5rt_execution_stream_operation_retain_output_port(op, "y", &out_port)` | `e5rt_execution_stream_*` | retain op 的具名输出端口 |
| `e5rt_io_port_bind_buffer_object(port, buf)` | `e5rt_io_port_*` | 把 buffer object 绑到具名端口 |

### 5.4 dispatch 阶段（family: `e5rt_execution_stream_*`）

| API | 用途 |
|---|---|
| `e5rt_execution_stream_create(&stream)` | 创建 execution stream |
| `e5rt_execution_stream_operation_prepare_op_for_encode(op)` | 每次 encode 前准备 op |
| `e5rt_execution_stream_encode_operation(stream, op)` | 把 op 编码进 stream |
| `e5rt_execution_stream_execute_sync(stream)` | 同步提交，阻塞直到完成 |
| `e5rt_execution_stream_submit_async(stream, ...)` | 轻量 async，返回 submit_id + complete_id |
| `e5rt_execution_stream_submit_async_with_timeout(stream, ..., timeout)` | 完整 async，带 error object + timeout（async 完成可能 hang） |
| `e5rt_execution_stream_reset(stream)` | 重置 stream 以便下一次 dispatch |

---

## 6. 端到端时序：编译 + 加载

```
┌─ 用户进程 ────────────────────────────────────────────────────────┐
│  // 步骤 ①：构图（用户自己负责）                                   │
│  write ".espresso.net" to disk                                    │
│                                                                   │
│  // 步骤 ②：compile                                                │
│  e5rt_e5_compiler_create_with_config(&compiler, config);          │
│  e5rt_e5_compiler_compile(compiler, "model.espresso.net",         │
│                            options, &library);                     │
│       │                                                           │
│       │  e5rt 内部 XPC 调 broker                                   │
└───────┼───────────────────────────────────────────────────────────┘
        │
        ▼
┌─ broker daemon ───────────────────────────────────────────────────┐
│  // 步骤 ②-a：检查 entitlement                                     │
│  copyClientEntitlement → com.apple.aned.private.allow?            │
│                                                                   │
│  // 步骤 ②-b：调 ANECompiler                                       │
│  ANECompiler(model_path, options) → program bytes + weights       │
│                                                                   │
│  // 步骤 ②-c：签名                                                  │
│  corecrypto sign(program_bytes)                                   │
│  写到 daemon-trusted 临时文件（vnode 进 trustcache）                │
│                                                                   │
│  // 步骤 ②-d：调 kernel 注册程序                                    │
│  open H11ANEInUserClient                                          │
│  → selector 0: ANE_DeviceOpen                                     │
│  → selector 3: ANE_ProgramCreate(program_image)                   │
│  → selector 4: ANE_ProgramPrepare(...)                            │
│                                                                   │
│  // 返回 program handle 给用户进程                                  │
└───────┬───────────────────────────────────────────────────────────┘
        │
        ▼
┌─ 用户进程（继续）──────────────────────────────────────────────────┐
│  // 步骤 ③：load（在 e5rt 层完成）                                  │
│  e5rt_program_library_retain_program_function(                    │
│      library, "fn_name", &function);                              │
│  e5rt_precompiled_compute_op_create_options_create_with_program_function(
│      &op_opts, function);                                         │
│  e5rt_execution_stream_operation_create_precompiled_compute_operation_with_options(
│      &op, op_opts);                                               │
│                                                                   │
│  // 步骤 ④：bind（在 e5rt 层完成）                                  │
│  e5rt_buffer_object_alloc(&buf, nbytes, 0);                       │
│  e5rt_execution_stream_operation_retain_input_port(op, "x", &p);  │
│  e5rt_io_port_bind_buffer_object(p, buf);                         │
│                                                                   │
│  // 步骤 ⑤：dispatch（进入热循环）                                  │
│  e5rt_execution_stream_create(&stream);                           │
│  for (;;) {                                                       │
│    fill_buf(buf);                                                 │
│    e5rt_execution_stream_operation_prepare_op_for_encode(op);     │
│    e5rt_execution_stream_encode_operation(stream, op);            │
│                                                                   │
│       │                                                           │
│       │  e5rt 内部 XPC 调 broker → selector 2                      │
└───────┼───────────────────────────────────────────────────────────┘
        │
        ▼
┌─ broker daemon ───────────────────────────────────────────────────┐
│  // 每次提交：转手 selector 2                                       │
│  H11ANEInUserClient::externalMethod(sel=2, args)                  │
│  （2376 in / 40 out 校验通过）                                     │
└───────┬───────────────────────────────────────────────────────────┘
        │
        ▼
┌─ 内核 driver ─────────────────────────────────────────────────────┐
│  ANE_ProgramSendRequest(client, ref, args)                        │
│   → ANEClientDevice::programSendRequest(ANEProgramRequestArgs*)   │
│   → ANEDriver::ANE_ProgramSendRequest(...)   // command-gate       │
│   → ANEHWDevice::doorBellRing(db)                                 │
│   → ANERegisterControl::write32(reg, 1 << idx)  // MMIO write     │
└───────┬───────────────────────────────────────────────────────────┘
        │
        ▼
┌─ ANE 引擎 ────────────────────────────────────────────────────────┐
│  mailbox 收到 doorbell → 调起 datapath → 跑程序                    │
│  完成后：固件清 sysreg bit 39 → store 到 host aperture → 关窗       │
│  → mach wake port callback → 回到用户进程                          │
└───────────────────────────────────────────────────────────────────┘
```

---

## 7. 一次回答三个关键问题

### Q1：编译在哪里发生？

**在 broker daemon 进程里**。用户进程通过 e5rt API 发起 `e5rt_e5_compiler_compile`，e5rt 内部 XPC 到 broker daemon，daemon 调它内部的 ANECompiler，产出 program bytes，签名后写到 daemon-trusted 临时文件，再通过 IOKit selector 3/4 注册到 kernel。

### Q2：加载在哪里发生？

**分两层**：
- **e5rt 层（用户进程）**：retain program function、构造 compute op options
- **kernel 层**：selector 3 `ANE_ProgramCreate` + selector 4 `ANE_ProgramPrepare` 实际把 program image 注册到 kernel 并准备运行资源；首次加载付出一次性成本产出 loadable hardware form

### Q3：用户进程为什么不直接调 IOKit？

**因为 `com.apple.ane.iokit-user-access` 全系统只有 broker daemon + 它的 per-user 兄弟持有**。任何 application 进程都不持有这个内核 entitlement，无法开 user client。所有应用必须经 broker，用 `com.apple.aned.private.*` 家族证明身份。这是 ch8 / ch27 反复强调的安全模型根。

> **唯一例外**：27 个二进制持有 `com.apple.security.temporary-exception.iokit-user-client-class`，可以跳过 broker 自投递（latency 优化，不是 capability 授予）。privileged device open 仍在 broker，broker 把 program handle + intermediate-buffer handle 交给这种客户端。

---

## 8. 三条最值得记住的判断

1. **对外暴露给用户的接口面只有一个：e5rt_* C API**——292 个 entry point，正常 dispatch 用 ≈52 个，五组 family（compile / load / bind / dispatch）。selector ABI、XPC、entitlement 全是 e5rt **内部**细节，用户看不见。

2. **编译和加载跨三个进程**：用户进程（e5rt）发起 → broker daemon（ANECompiler + 签名 + IOKit 客户端）执行 → kernel（IOKit selector 0/3/4）注册。**用户进程碰不到 IOKit**——entitlement 把 device-open 压到全系统 2 个二进制。

3. **selector 2 是 user-space 到 hardware doorbell 的唯一入口**——2376 in / 40 out，经四层下降到 MMIO write。所有 dispatch（不论是 user 通过 e5rt，还是 daemon 直接调）最终都收敛到 selector 2 + doorbell。

---

## 9. 与其他章节的勾连

- **→ 第5章（Software stack）**：本文的四个进程边界对应 ch5 的软件栈分层
- **→ 第6章（Dispatching without Core ML）**：本文的 e5rt 接口表 = ch6 Listing 6.1 + Table 6.1 的展开
- **→ 第7章（Weights and compression）**：编译阶段的权重布局（streaming vs fold）发生在 daemon 内的 ANECompiler
- **→ 第8章（Entitlement boundary）**：本文的"用户进程 ↔ daemon"接口 = ch8 的"delegate signing"步骤实例化
- **→ 第22章（Compiler）**：daemon 内的 ANECompiler 详见 ch22
- **→ 第23章（Program and container format）**：program image 三阶段（bundle → image → firmware container）详见 ch23
- **→ 第25章（Compression internals）**：编译期的 scale/bias 折叠、OCG 打包、relocation 都发生在 daemon 内的 ANECompiler
- **→ 第26章（Hidden layers & netplist）**：用户写 `.espresso.net` 就是给 e5rt compiler API 的 `model_path` 参数
- **→ 第27章（Kernel driver and IOKit ABI）**：本文的 daemon ↔ kernel 接口表 = ch27 selector 表
- **→ 第28章（DART）**：bind 阶段的 buffer object 在 kernel 侧经 selector 5 `ANE_MemoryMapRequest` 走 DART
- **→ 第29章（Firmware）**：program loader 解析 magic 0xbeefface 见 ch29
- **→ 第30章（Host-to-firmware command protocol）**：doorbell write 详见 ch30

---

*本文档基于 arXiv:2606.22283v1 (21 Jun 2026) 跨章整合（ch5 / ch6 / ch8 / ch26 / ch27）撰写。论文为逆向工程参考文档，所描述的私有接口未公开、未受支持、跨版本脆弱，仅供研究、测量和设备端实验使用。Core ML 仍是唯一受支持的发行路径。*
