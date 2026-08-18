# Qwen3.8-27B + OMP Vibe：一次本地 Agent 部署实验

这是一次面向日常编码工作的个人部署记录：在 WSL2 + RTX 3090 上保留本地 Qwen3.8-27B 4-bit 推理能力，同时比较远端 DeepSeek Flash、远端 Codex 与本地 Qwen 组成的 OMP Vibe Agent 路由。

## 结论先行

当前默认采用 **All-DeepSeek Flash + 自然 Vibe**。本地 Qwen 服务保留为显式按需启动的能力，而不是默认编排路径。

在一个相同的 Matplotlib `Axes.clear()` 修复任务上，纯 DeepSeek 自然 Vibe 完成并通过独立验证，端到端用时 344.4 秒；DeepSeek Director + 本地 Qwen worker 的自然 Vibe 记录用时 1495.8 秒。后者没有形成有效并行，反而引入了本地推理等待和更长的工作链。

这不是对所有任务、模型或硬件的普适排序。它说明：当远端 Director 已有足够的执行能力、任务存在依赖链且本地 worker 没能并行交付时，混合路由不一定带来实际收益。

完整结果与方法见 [REPORT.md](REPORT.md) 和 [METHODOLOGY.md](METHODOLOGY.md)。

## 发布范围

本仓库只包含脱敏的实验叙述、聚合指标和方法说明。它**不包含**：

- 模型权重、量化文件或 Python 环境；
- OMP overlay、服务脚本或任何可执行代码；
- 原始 session JSONL、终端日志、工作区快照或数据集副本；
- API 密钥、令牌、个人路径或可识别信息。

## 日用策略

| 使用场景 | 路由 |
| --- | --- |
| 默认交互与自然 Vibe | DeepSeek Flash 全远端 |
| 需要本地模型试验或离线执行 | 显式启动本地 Qwen 服务 |
| 追求最高复杂推理质量 | 显式使用 Codex |

本地模型并没有被删除。它保留用于离线任务、后端实验和需要本地 worker 的特定工作流；只是不再让它成为默认路径。

## 许可

文档以 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 发布。
