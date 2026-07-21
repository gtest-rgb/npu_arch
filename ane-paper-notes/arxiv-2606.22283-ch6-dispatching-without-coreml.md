# 第6章《Dispatching without Core ML》笔记

> 论文：*Apple Neural Engine: Architecture, Programming, and Performance*
> 作者：Spencer H. Bryngelson（Georgia Institute of Technology）
> 提交：2026-06-21，302 页，CC-BY 4.0
> 本章原文：pp. 37–40
>
> 本章是直连路径的"用户侧权威说明书"。回答三个问题：**execution runtime 在用户进程里的角色**、**对外暴露的接口**、**直连路径调哪些接口编译和加载**。

---

## 1. execution runtime 到底是什么

之前在 ch8 精炼版里把 execution runtime 笼统说成"daemon + kernel + 引擎"——**这个定义不精确**。读完 ch6 后真正定义如下：

> **execution runtime = 用户进程内的 C 分发层**，即 `e5rt_*` 函数族，从 Espresso framework 导出。

它是用户进程里的**一组 C API**，负责把"用户写好的网络描述"驱动到"engine 跑出结果"。所有跨进程（去 daemon）、跨内核（去 driver）的细节都被这层 C API 包住。

```
┌──────────────────────────────────────────────────────────────┐
│  用户进程                                                     │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  execution runtime（e5rt_* C API 层）                  │  │
│  │  · Espresso framework 导出                              │  │
│  │  · 292 个 entry point（≈275 可调，≈52 走正常 dispatch）│  │
│  │  · 所有调用返回 int64_t 错误码                          │  │
│  └────────────────────────┬───────────────────────────────┘  │
│                           │                                  │
└───────────────────────────┼──────────────────────────────────┘
                            │  内部经 IOKit/Mach XPC 越界
                            ▼
                       daemon / kernel / engine
                       （execution runtime 之外）
```

**两个不变量**：
- 每个返回 int64_t，0 表示成功
- 创建对象的 API 通过第一个 out-param 返回，compile/retain 通过最后一个

---

## 2. 直连路径五步法

ch6 §6.1 给出五步，对应 e5rt_* 的五组调用：

```
┌─────────────────────────────────────────────────────────────┐
│  ① Build   构图：layer-and-wiring graph，含 op / 参数 /      │
│              权重 blob / 输入输出端口                          │
│  ② Compile 编译：runtime 驱动编译器，lowering + 验证 +       │
│              把权重按 streaming datapath 摆好，写盘到         │
│              content-addressed cache（一次付费）              │
│  ③ Load    加载：从 cache 打开编译产物，实例化它暴露的函数，  │
│              首次加载付出一次性成本产出 loadable hardware form │
│  ④ Bind    绑定：每个外部 I/O 端口绑定一个 buffer            │
│  ⑤ Dispatch 分发：把 loaded program 编为 execution stream    │
│              里的一个 op，同步或异步提交                       │
└─────────────────────────────────────────────────────────────┘
```

**热循环只有 ④ + ⑤**。① ② ③ 是一次性付费，在循环外。

---

## 3. 接口目录（Listing 6.1 + Table 6.1）

### 3.1 按阶段分组

| Family | 阶段 | 代表 API |
|---|:-:|---|
| `e5rt_e5_compiler_*` | ② compile | `create_with_config` / `compile` / `is_new_compile_required` |
| `e5rt_program_library_*` / `e5rt_program_function_*` | ③ load | `create` / `retain_program_function` / `load_for_execution` |
| `e5rt_precompiled_compute_op_*` | ③ load | `create_options_create_with_program_function` |
| `e5rt_buffer_object_*` / `e5rt_io_port_*` | ④ bind | `alloc` / `get_data_ptr` / `bind_buffer_object` |
| `e5rt_execution_stream_*` | ⑤ dispatch | `encode_operation` / `execute_sync` / `submit_async` / `reset` |

### 3.2 完整调用序列（论文 Listing 6.1 原版）

```c
/* ② Compile —— runtime 驱动 out-of-process compiler */
e5rt_e5_compiler_create_with_config(&compiler, config);
e5rt_e5_compiler_compile(compiler, model_path, options, &library);

/* ③ Load —— retain 程序暴露的可调用函数 */
e5rt_program_library_retain_program_function(library, fn_name, &function);
e5rt_precompiled_compute_op_create_options_create_with_program_function(
    &op_opts, function);
e5rt_execution_stream_operation_create_precompiled_compute_operation_with_options(
    &op, op_opts);

/* ④ Bind —— 给每个具名 I/O 端口绑一个 CPU buffer object */
e5rt_buffer_object_alloc(&buf, nbytes, /*type=*/0);
e5rt_execution_stream_operation_retain_input_port(op, "x", &in_port);
e5rt_io_port_bind_buffer_object(in_port, buf);

/* ⑤ Dispatch —— 热循环里 encode + submit */
e5rt_execution_stream_create(&stream);
for (;;) {
    /* 通过 e5rt_buffer_object_get_data_ptr(buf, &ptr) 填输入 */
    e5rt_execution_stream_operation_prepare_op_for_encode(op);
    e5rt_execution_stream_encode_operation(stream, op);
    e5rt_execution_stream_execute_sync(stream);  /* 或 submit_async */
    /* 通过 data pointer 读回输出 */
    e5rt_execution_stream_reset(stream);
}
```

### 3.3 编译选项：string-keyed dict

编译选项**不是定长 struct**——是 `std::any` 值的字符串键字典：

| 键名 | 值类型 | 例子 |
|---|---|---|
| `forceRecompilation` | `bool` | 强制重编 |
| `segmenter` | `std::string` | 选 segmenter |
| `computeDeviceTypesAllowed` | `std::vector<ComputeDeviceType>` | 设备 mask，`0x4` = ANE |

> **含义**：加一个选项不会改 option handle 的固定字节偏移——所以逆向时不能"按 offset 推字段"。

---

## 4. 直连路径"性能完备"的根（§6.4）

ch6 给出一个强论断：**直连路径在 dispatch 上"性能完备"**——它达到任何更高特权路径的吞吐。

### 4.1 单次提交可驱动多个 on-engine 步

- 同一次 submission 可以驱动 engine 上多个步骤，**中间不需要 host 往返**
- 机制：第2章的 resident state——一个 buffer 跨步骤留在 engine working set 里，前一步输出 = 下一步输入（in place）
- host 只在每个步骤里送"少量 per-step 输入"，到 checkpoint 才读回 held buffer
- **省掉的是"每步两次跨边界复制全状态"，不是省 host 调用次数本身**

### 4.2 算术验证

- 单次提交驱动 N 步的速率 = N 次 host 调用每步一次的速率
- **去掉 host 调用不会加速**——engine 才是瓶颈，不是 host dispatch
- 因此 unentitled direct path 在 dispatch 上**性能完备**

### 4.3 stream 的三种提交形式

| 形式 | 行为 |
|---|---|
| synchronous | 阻塞调用者直到 stream 完成 |
| lightweight async | 返回 submit_id + complete_id |
| full async | 带 error object + timeout 参数（timeout 存在是因为 async 完成可能 hang） |

**stream 并发的坑**：
- **单 stream 流水**安全：保持多个 op 在飞，每个带自己的 completion event，按 event 排干
- **跨 stream overlap 不安全**：第 2+ 个 stream 的 completion event 永远不通知，等待者阻塞
- 破阻塞的控制：`low-latency-async-event` stream option + `timeout` 参数

---

## 5. 五步 → 两阶段（§6.5）

实战中五步折成两阶段：

```
┌──────────── 一次性、循环外 ───────────┐
│  Build network description           │
│  Compile to program format           │
│  Load program                        │
└──────────────────┬───────────────────┘
                   │
                   │  端口已通过 io_port_bind_buffer_object 绑好
                   │
┌──────────────────▼───────────────────┐
│  Per frame / token / request ──────┐ │
│    Bind operand buffers             │ │
│    Dispatch                         │ │  ← 热循环
│    Read outputs                     │ │
│    Reset stream                     │ │
│  ─────────────────────────────────  │ │
└──────────────────────────────────── ┘ ┘
```

简化后的热循环：

```c
/* 循环外：compile + load + bind（一次性） */
e5rt_e5_compiler_compile(compiler, model_path, options, &library);
e5rt_program_library_retain_program_function(library, fn_name, &function);
e5rt_precompiled_compute_op_create_options_create_with_program_function(&op_opts, function);
e5rt_execution_stream_operation_create_precompiled_compute_operation_with_options(&op, op_opts);
e5rt_execution_stream_create(&stream);
/* io_port_bind_buffer_object 已绑好 */

for (;;) {
    e5rt_execution_stream_operation_prepare_op_for_encode(op);
    e5rt_execution_stream_encode_operation(stream, op);
    e5rt_execution_stream_execute_sync(stream);
    e5rt_execution_stream_reset(stream);
}
```

---

## 6. Netplist 是 execution runtime 的"一等公民"

§6.1 末尾的论述值得记下：

> "A network expressed for the compiler is a netplist, the `.espresso.net` representation the compiler accepts **alongside `.mil`**."

含义：
- **execution runtime 不挑输入来源**——你给它 `.mlpackage` / `.mil` / `.espresso.net` 都行
- 但 netplist 是**直连路径的标准输入**，因为它直接描述图结构，不经过 Core ML 转换器
- netplist 的 schema 由 binary 内部恢复出的字典字符串常量决定（参见 ch22 + ch26）

---

## 7. 三条最值得记住的判断

1. **execution runtime 是用户进程里的 C API 层（`e5rt_*`），不是 daemon 或 kernel**——之前 ch8 精炼版的笼统定义应该按 ch6 修正为"用户进程的 C dispatch 层 + 它跨进程跨内核调用的 daemon/kernel/engine"。

2. **直连路径接口共 292 个 entry point，正常 dispatch 只用 ≈52 个**——核心是 Listing 6.1 的 5 步调用序列（compiler_create + compile / retain_function + create_compute_op / buffer_alloc + bind / encode + execute + reset）。编译加载的接口就在这张表里。

3. **直连路径"性能完备"**——单次 submission 可驱动多个 on-engine 步、engine 才是瓶颈不是 host，所以 unentitled 路径吞吐 = 最高特权路径吞吐。Entitlement 多的是 framework 便利性，不是引擎算力。

---

## 8. 回答最初的问题

> **用户进程中的 execution runtime 的作用？**
> 它是 `e5rt_*` C dispatch 层（Espresso framework 导出），把用户写好的网络描述驱动到 engine 跑出结果。292 个 entry point，覆盖 compile / load / bind / dispatch 全流程。
>
> **对外提供给用户的接口？**
> 五组 family（见 §3.1）：`e5rt_e5_compiler_*` / `e5rt_program_library_*` / `e5rt_precompiled_compute_op_*` / `e5rt_buffer_object_*` + `e5rt_io_port_*` / `e5rt_execution_stream_*`。
>
> **直连路径调哪些接口进行编译和加载？**
> 编译：`e5rt_e5_compiler_create_with_config` → `e5rt_e5_compiler_compile`（runtime 内部去 daemon 跑 out-of-process compiler）。
> 加载：`e5rt_program_library_retain_program_function` → `e5rt_precompiled_compute_op_create_options_create_with_program_function` → `e5rt_execution_stream_operation_create_precompiled_compute_operation_with_options`。
> 绑定：`e5rt_buffer_object_alloc` → `e5rt_execution_stream_operation_retain_input_port` → `e5rt_io_port_bind_buffer_object`。
> 分发：`e5rt_execution_stream_create` → 循环里 `prepare_op_for_encode` + `encode_operation` + `execute_sync` + `reset`。

---

## 9. 与其他章节的勾连

- **→ 第5章（Software stack）**：execution runtime 在第5章定义的软件栈里的位置
- **→ 第8章（Entitlement boundary）**：本章定义的 e5rt_* 直连路径**不触碰 entitlement gate**——见 ch8 §8.3。ch8 精炼版里"execution runtime = daemon + kernel + engine"应按本章修正
- **→ 第22章（Compiler）**：netplist schema 的每个 key 在 ch22 详列
- **→ 第26章（Hidden layers & netplist）**：ch26 的 netplist 直写就是本章第①步 build 的具体玩法，触发 ch26 §26.3 的隐藏 descriptor
- **→ 第27章（Kernel driver and IOKit ABI）**：本章 e5rt_* 在用户态，它跨进程/跨内核到达 ch27 的 IOKit selector 和 doorbell
- **→ 第2章（Execution model）**：§6.4 的"resident state 跨步驻留"是第2章 host-coprocaster 模型的展开

---

*本文档基于 arXiv:2606.22283v1 (21 Jun 2026) 第6章 (pp. 37–40) 撰写。论文为逆向工程参考文档，所描述的私有接口未公开、未受支持、跨版本脆弱，仅供研究、测量和设备端实验使用。Core ML 仍是唯一受支持的发行路径。*
