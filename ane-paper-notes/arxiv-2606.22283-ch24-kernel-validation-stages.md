# 内核校验的三个时刻专题

> 论文：*Apple Neural Engine: Architecture, Programming, and Performance*
> 作者：Spencer H. Bryngelson（Georgia Institute of Technology）
> 提交：2026-06-21，302 页，CC-BY 4.0
> 关联章节：ch6《Dispatching without Core ML》、ch24《HAL and capability gates》、
> ch27《Kernel driver and IOKit ABI》
>
> 回答一个具体问题：在 execute_sync → aned → 内核 → 硬件 这条路径里，
> **内核的校验在哪一步**？

---

## 0. 结论先行

校验**不是一处**，而是分布在**三个时刻**。execute_sync 流程里能看到的只是
**最后一个**——前面两轮早在 compile / load 阶段就已完成。

| 时刻 | 谁做 | 哪个进程/层 | 校验什么 |
|---|---|---|---|
| ① Compile time | ANECompilerService | 守护进程（编译器） | `MinimumFamily` trait / netplist schema / 生成签名 |
| ② Load time | selector 3/4 handler | 内核 user client | 签名验证 / 结构完整性 / 资源登记 |
| ③ Exec time | selector 2 handler | 内核 user client + driver | entitlement / inputCnt / **HAL capability bytes** |

---

## 1. 三轮校验套回直连路径五步法

```
② Compile:    e5rt_e5_compiler_compile
                    │
                    │ 跨进程到 ANECompilerService
                    ▼
              ┌──────────────────────────────────────────┐
              │  ⚠ 校验 ①：MinimumFamily trait            │
              │     · netplist 里 Target = "H13"         │
              │       vs 芯片 HAL 表里列出的原生 op       │
              │     · 缺 op 的 → 编译失败（编译器就拒）   │
              │  ⚠ 校验：netplist schema 1.0.10          │
              │  ⚠ 生成签名（写 content-addressed cache） │
              └──────────────────────────────────────────┘


③ Load:       e5rt_program_library_retain_program_function
                    │
                    │ aned 代调 selector 3 ANE_ProgramCreate
                    │            selector 4 ANE_ProgramPrepare
                    ▼
              ┌──────────────────────────────────────────┐
              │  ⚠ 校验 ②：程序签名验证                   │
              │     · 公钥验签（防篡改）                  │
              │     · 签名链 anchor 必须在 trust cache    │
              │  ⚠ 校验：结构完整性                       │
              │     · program 字节布局合法                │
              │     · 资源（缓存/DMA 通道）可分配         │
              │  → 通过则给 program id（后面 selector 2 用） │
              └──────────────────────────────────────────┘


⑤ Exec:       execute_sync
                    │
                    │ e5rt → Mach IPC → aned → selector 2
                    ▼
              ┌──────────────────────────────────────────┐
              │  ⚠ 校验 ③a（user client 层）：            │
              │     · inputCnt == 2376 ？                │
              │       否 → kIOReturnBadArgument           │
              │                  (0xe00002c2)             │
              │     · 调用者持                            │
              │       com.apple.ane.iokit-user-access ？ │
              │       否 → kIOReturnNotPrivileged         │
              │     · selector 号在 0–16 范围内 ？        │
              └──────────────────────────────────────────┘
                    │
                    ▼
              ┌──────────────────────────────────────────┐
              │  ⚠ 校验 ③b（ANEClientDevice 层）：        │
              │  ┌────────────────────────────────────┐ │
              │  │ HAL capability bytes gate          │ │
              │  │ （ch24 §24.3 的运行时落点）         │ │
              │  │                                    │ │
              │  │  · 当前芯片的 HAL 表 vs program     │ │
              │  │    引用的 op 集合                   │ │
              │  │  · 任何不匹配 →                     │ │
              │  │    kIOReturnUnsupported            │ │
              │  │    (0xe00002c7)                    │ │
              │  └────────────────────────────────────┘ │
              │  · program id 在 ② 登记过 ？            │
              │  · buffer 都经过 selector 5 映射 ？     │
              │  · 引擎未关闭 → 否则 0xe00002c5         │
              └──────────────────────────────────────────┘
                    │
                    ▼
              ANEDriver → ANEHWDevice → doorbell → 硬件
```

---

## 2. 三轮校验各自的"通过 ≠ 可达"含义

这是 ch26 反复强调的 **`validation-passes ≠ reachable`**：

| 校验时刻 | 通过意味着 | 不意味着 |
|---|---|---|
| ① Compile (`MinimumFamily` trait) | "编译器以为这个 floor 能跑" | 真正的芯片表里有这个 op |
| ② Load (selector 3/4) | "程序签名和结构合法" | op 在硬件上真的存在 |
| ③ Exec (HAL capability bytes) | **"这块芯片 HAL 表里真有这个 op"** ← 真正的硬门 | — |

只有 ③ 通过，op 才真的能跑。① ② 是**程序级校验**，③ 是**硬件级 attestation**。

---

## 3. ch24 的两套 gate 落点

| Gate | 何时 | 何层 |
|---|---|---|
| `MinimumFamily` trait | ① Compile time | 编译器守护进程 |
| HAL capability bytes | ③ Exec time | 内核 `ANEClientDevice::programSendRequest` |

**两套 gate 互不一致**——这是 ch24 的核心论断。
编译器可能认为合法（trait 通过），但运行时内核 HAL 表拒绝
（`0xe00002c7`）。这就是为什么 ch26 说"必须分别通过"。

---

## 4. 在 execute_sync 流程里的精确位置

回到 ch27 selector 2 的四层 MMIO 调用链，校验落在 **⑦ 步和 ⑧ 步**：

```
⑥  IOUserClient::externalMethod         ← mach trap 落点
       │
       ▼
⑦  ANE_ProgramSendRequest handler       ← 校验 ③a：input size + entitlement
       │
       ▼
⑧  ANEClientDevice::programSendRequest  ← 校验 ③b：HAL capability bytes
       │                                  ← ch24 §24.3 真正的硬门
       ▼
⑨  ANEDriver                            ← 不再校验，纯翻译
⑩  ANEHWDevice::doorBellRing
⑪  ANERegisterControl::write32          ← 单次 MMIO 写
```

- **校验 ③a 是 ABI 契约层**（字节大小、entitlement）
- **校验 ③b 是能力语义层**（HAL 表里的 op 是否存在）

两者都在 user client 之后、driver 翻译之前——
**doorbell 写之前必须全部通过**，否则不消耗硬件资源。

---

## 5. 失败码落点对照

| 失败码 | 触发位置 | 校验阶段 |
|---|---|---|
| `0xe00002c2` (`kIOReturnBadArgument`) | ⑦ | input 结构不合法 |
| `kIOReturnNotPrivileged` | ⑦ | entitlement 缺失 |
| `0xe00002c7` (`kIOReturnUnsupported`) | ⑧ | **HAL capability gate 拒绝** |
| `0xe00002c5` (closed/closing) | ⑧ | 引擎状态不对 |

---

## 6. 与各章的勾连

- **→ 第6章（Dispatching without Core ML）**：execute_sync 是五步法第 ⑤ 步 Dispatch 的
  同步变体；三轮校验中的 ③ 就是这一步触发的内核校验
- **→ 第24章（HAL and capability gates）**：本文把 ch24 的两套 gate 落到具体调用链的
  位置——`MinimumFamily` 在编译期、HAL capability bytes 在 selector 2 的
  `ANEClientDevice` 层
- **→ 第26章（Hidden layers & netplist）**：本文解释了 ch26 强调的
  "validation-passes ≠ reachable" 在调用栈上对应的具体位置
- **→ 第27章（Kernel driver and IOKit ABI）**：本文展开 selector 2 handler 内部的
  两层校验（user client 层 + driver 层），对应 ch27 §27.3 的四层 MMIO 调用链
- **→ 跨章专题（直连路径：编译与加载的端到端流程）**：本文是该专题里
  "执行阶段的内核校验"切片

---

## 7. 一句话总结

> **校验分三轮**：编译时（`MinimumFamily` trait，编译器守护进程）、
> 加载时（签名 + 结构，selector 3/4 在内核）、执行时（HAL capability bytes，
> selector 2 的 `ANEClientDevice` 层）。
> 前面两轮已在前置流程完成；execute_sync 流程里看到的是**第三轮**，
> 落在 selector 2 调用链的 ⑦ user client 和 ⑧ ANEClientDevice 两步。

---

*本文档基于 arXiv:2606.22283v1 (21 Jun 2026) 第6章 (pp. 37–40)、
第24章、第27章 (pp. 177–185) 撰写。论文为逆向工程参考文档，所描述的私有接口
未公开、未受支持、跨版本脆弱，仅供研究、测量和设备端实验使用。
Core ML 仍是唯一受支持的发行路径。*
