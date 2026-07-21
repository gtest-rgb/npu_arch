# `e5rt_program_library_retain_program_function` 语义专题

> 论文：*Apple Neural Engine: Architecture, Programming, and Performance*
> 作者：Spencer H. Bryngelson（Georgia Institute of Technology）
> 提交：2026-06-21，302 页，CC-BY 4.0
> 关联章节：ch6《Dispatching without Core ML》（pp. 37–40）
>
> 回答一个具体问题：直连路径五步法里 ③ Load 阶段出现的
> `e5rt_program_library_retain_program_function(library, fn_name, &function)`
> 到底做什么。

---

## 1. "retain" 是什么

`retain` 是 Objective-C / Core Foundation / C++ 引用计数体系里的标准动词，意思是
"**增加一次引用计数、拿到对象句柄**"。在 e5rt 这套 C API 里，它对应一对操作：
`retain_*` / `release_*`，语义类似 `CFRetain` / `CFRelease`。

```c
e5rt_program_library_retain_program_function(library, fn_name, &function);
```

- **输入**：`library`（compile 出来的程序库句柄）+ `fn_name`（程序库里某个具名函数的名字）
- **输出**：通过 out-param 返回一个 `function` 句柄（指向库内那一个可调用函数）
- **副作用**：把 library 内部那个 function 对象的引用计数 +1。你不 release，库就不会被回收；
  release 到 0，底层资源才释放

---

## 2. 为什么是 retain 而不是 create

编译产物 `library` 是一个**容器**，里面可能暴露多个函数：一个网络常常有多个入口
（encoder/decoder、不同 batch、不同精度变体、不同输入组合……）。
retain 表达的语义是：

> "library 里本来就有这个 function 对象，我只是要把它**取出来并保住**，
> 不让它被回收，而不是新建一个。"

这与前面 compile 阶段用 `create_*`（真造一个新对象）是不同的语义层级：

| 动词 | 语义 | 例子 |
|---|---|---|
| `create_*` | 新建对象（分配资源、填充字段、计数置 1） | `e5rt_e5_compiler_create_with_config` |
| `retain_*` | 引用计数 +1，返回已有对象的句柄 | `retain_program_function` |
| `release_*` | 引用计数 −1，减到 0 时回收 | 对应的 release |

---

## 3. 它在 ch6 五步法里的位置

```
② Compile:  e5rt_e5_compiler_compile  → library              （容器）
    ↓
③ Load:     e5rt_program_library_retain_program_function → function
                                                                    ↑ 取出函数句柄
            e5rt_precompiled_compute_op_create_options_create_with_program_function(&op_opts, function)
            e5rt_execution_stream_operation_create_precompiled_compute_operation_with_options(&op, op_opts)
    ↓
④ Bind:     e5rt_buffer_object_alloc / retain_input_port / bind_buffer_object
    ↓
⑤ Dispatch: e5rt_execution_stream_create / encode_operation / execute_sync / reset
```

拿到 `function` 句柄之后，下一步用它构造一个**可被 dispatch 的 operation**
（封装成 execution stream 里的一个算子）。`function` 是"函数指针"，
`op` 是"调用现场"。

---

## 4. 为什么 retain 不触发内核调用

这一步**完全发生在用户进程的 e5rt C 层**，不会再触发 selector 3/4。
原因是：

- **selector 3 (`ANE_ProgramCreate`) / selector 4 (`ANE_ProgramPrepare`)**
  在 compile 阶段已经跑过了——daemon 把签名程序注册进内核、把硬件资源
  （缓存分块、DMA 描述符等）准备好，返回一个内核侧的 program id
- compile 完成后，`library` 句柄内部就已经持有了那个 program id
- `retain_program_function` 只是把库内某个入口的**用户态引用**保住，
  不重新进内核

换句话说：
**compile 阶段已经把内核侧的状态机推到位了；Load 阶段剩下的 retain / create_op
只是在用户进程里把"怎么调度这个已注册 program"的句柄搭好**。

---

## 5. 类比

便于记忆的 mental model：

| e5rt 对象 | 类比 |
|---|---|
| `library` | 一个 `.dylib`（动态库） |
| `function` | 用 `dlsym(lib, "fn_name")` 拿到的函数指针 |
| `retain_program_function` | 把这个符号的引用保住，避免库被卸载后符号悬空 |
| `op` | 把函数指针 + 绑定的参数，打包成一次可调用的 invocation |

---

## 6. 回到 Listing 6.1 的上下文

把这一步放回论文 Listing 6.1 的调用序列：

```c
/* ② Compile */
e5rt_e5_compiler_create_with_config(&compiler, config);
e5rt_e5_compiler_compile(compiler, model_path, options, &library);
                                                    /* ↑ library 出来 */

/* ③ Load —— retain function 在这一步起手 */
e5rt_program_library_retain_program_function(library, fn_name, &function);
                                                    /* ↑ function 出来 */
e5rt_precompiled_compute_op_create_options_create_with_program_function(
    &op_opts, function);
e5rt_execution_stream_operation_create_precompiled_compute_operation_with_options(
    &op, op_opts);

/* ④ Bind */
e5rt_buffer_object_alloc(&buf, nbytes, /*type=*/0);
e5rt_execution_stream_operation_retain_input_port(op, "x", &in_port);
e5rt_io_port_bind_buffer_object(in_port, buf);

/* ⑤ Dispatch */
e5rt_execution_stream_create(&stream);
for (;;) {
    e5rt_execution_stream_operation_prepare_op_for_encode(op);
    e5rt_execution_stream_encode_operation(stream, op);
    e5rt_execution_stream_execute_sync(stream);
    e5rt_execution_stream_reset(stream);
}
```

注意第 4 步 `retain_input_port` 用的也是 retain——同一个动词家族，
同一个"把已有对象句柄保住"的语义。

---

## 7. 一句话总结

> **retain function = 从编译产物里取一个具名入口的句柄，并 +1 引用计数，
> 后面用它构造可分发的 op**。它只在用户进程的 e5rt C 层完成，不进内核；
> 内核侧的 program registration 在更早的 compile 阶段已经通过 selector 3/4 完成。

---

## 8. 与其他章节的勾连

- **→ 第6章（Dispatching without Core ML）**：本文是对 ch6 Listing 6.1 第 ③ 步的展开
- **→ 第27章（Kernel driver and IOKit ABI）**：解释了为什么 retain 这一步**不**对应
  selector 3/4——内核侧的 program 生命周期管理已在 compile 阶段完成
- **→ 第8章（Entitlement boundary）**：retain 这一步不触碰 entitlement gate，
  因为它不跨进程；跨进程的编译在更早阶段已经由被授权的 daemon 完成
- **→ 跨章专题（直连路径：编译与加载的端到端流程）**：本文是该专题里
  "③ Load" 阶段的细节切片

---

*本文档基于 arXiv:2606.22283v1 (21 Jun 2026) 第6章 (pp. 37–40) 撰写。
论文为逆向工程参考文档，所描述的私有接口未公开、未受支持、跨版本脆弱，
仅供研究、测量和设备端实验使用。Core ML 仍是唯一受支持的发行路径。*
