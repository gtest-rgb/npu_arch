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

## 两条互补的 reachability 视角

- **第8章（软件门控）**：entitlement / framework loader / daemon 编译签名——回答"绕过 Core ML 的直连路径能到哪"
- **第24章（硬件 attested）**：HAL 表 + MinimumFamily trait——回答"一块芯片表里声明支持哪些 op"

两者共同构成 ANE 的能力边界判据：**attested ⊋ reachable**，软件 entitlement 无法撼动硬件 primitive 缺失。

## 第 25 章主线

权重压缩的四套 codec（int8 仿射 / int4 LUT / 结构化稀疏 / 块级仿射）+ 流式 vs 折叠两道开关 + 14 字 OCG 打包记录 + 双稀疏机制 + FP8/Winograd 边界情况。回答的是"权重字节怎么从训练好的张量变成芯片能直接吃的压缩流，以及谁在门控这条路径"。

## 免责声明

论文中描述的 direct route 未被 Apple 官方支持、跨版本脆弱，仅适用于研究、测量和设备端实验，不应用于发布软件。Core ML 仍是唯一受支持的发行路径。
