# ANE Paper Notes

深度解读笔记，基于 arXiv:2606.22283v1
[*Apple Neural Engine: Architecture, Programming, and Performance*](https://arxiv.org/abs/2606.22283)
（Spencer H. Bryngelson, Georgia Institute of Technology, 2026-06-21, 302 pages, CC-BY 4.0）。

论文是 ANE (Apple Neural Engine) 的逆向工程参考文档，覆盖 A11–A18 / M1–M5 全 chip family。

## 已解读章节

| 章节 | 标题 | 笔记 |
|---|---|---|
| 8 | Entitlement boundary | [arxiv-2606.22283-ch8-entitlement-boundary.md](./arxiv-2606.22283-ch8-entitlement-boundary.md) |
| 24 | HAL and capability gates | [arxiv-2606.22283-ch24-hal-capability-gates.md](./arxiv-2606.22283-ch24-hal-capability-gates.md) |
| 25 | Compression internals | [arxiv-2606.22283-ch25-compression-internals.md](./arxiv-2606.22283-ch25-compression-internals.md) |
| 26 | Hidden layers & direct netplist authoring | [arxiv-2606.22283-ch26-hidden-layers-netplist.md](./arxiv-2606.22283-ch26-hidden-layers-netplist.md) |

## 两条互补的 reachability 视角

- **第8章（软件门控）**：entitlement / framework loader / daemon 编译签名——回答"绕过 Core ML 的直连路径能到哪"
- **第24章（硬件 attested）**：HAL 表 + MinimumFamily trait——回答"一块芯片表里声明支持哪些 op"

两者共同构成 ANE 的能力边界判据：**attested ⊋ reachable**，软件 entitlement 无法撼动硬件 primitive 缺失。

## 第 25 章主线

权重压缩的四套 codec（int8 仿射 / int4 LUT / 结构化稀疏 / 块级仿射）+ 流式 vs 折叠两道开关 + 14 字 OCG 打包记录 + 双稀疏机制 + FP8/Winograd 边界情况。回答的是"权重字节怎么从训练好的张量变成芯片能直接吃的压缩流，以及谁在门控这条路径"。

## 第 26 章主线

**45 个原生 descriptor vs 框架层 ≈190 种 op**。手写 `.espresso.net`（schema 1.0.10）可以跳过 Core ML 翻译器，直接触达 Zin IR 里那 45 个原生描述符——其中藏着 SDPA、Sort、点云、流式状态、33 节 LUT 等"隐藏算子"。两套 gate（`MinimumFamily` trait vs HAL capability bytes）互不一致，必须分别通过；**验证通过 ≠ 可达**。SDPA 的 `SubtractMax` 标志是 fp16 数值正确性的命门。这条路是私有 API + 反编译得来，仅供研究测量，Core ML 仍是唯一发行路径。

## 免责声明

论文中描述的 direct route 未被 Apple 官方支持、跨版本脆弱，仅适用于研究、测量和设备端实验，不应用于发布软件。Core ML 仍是唯一受支持的发行路径。
