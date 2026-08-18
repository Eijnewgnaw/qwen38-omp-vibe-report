# Qwen3.8-27B + OMP Vibe：本地编码智能体部署记录

这是一次面向日常编码工作的个人部署实验：在 WSL2 和 RTX 3090 上保留本地 Qwen3.8-27B 4-bit 推理能力，同时比较远端 DeepSeek Flash、远端 Codex 与本地 Qwen 组成的 OMP Vibe 路由。

## 结论

当前默认采用 **全 DeepSeek Flash + 自然 Vibe**。本地 Qwen 服务保留为显式按需能力，而不再是默认编排路径。

在同一个 Matplotlib `Axes.clear()` 修复任务上，纯 DeepSeek 自然 Vibe 完成并通过独立复核，端到端用时 344.4 秒；DeepSeek Director 与本地 Qwen worker 组成的自然 Vibe 用时 1495.8 秒。后者没有形成有效并行，反而引入本地推理等待与更长的工作链。

这不是对所有模型、任务或硬件的普适排名。它说明：当 Director 已有足够执行能力、任务存在依赖链、而本地 worker 未能缩短关键路径时，混合路由未必有实际收益。

## 文档

- [实验报告](REPORT.md)
- [本地 Qwen 部署、框架、并发与上下文探索](LOCAL_QWEN_DEPLOYMENT.md)
- [方法与可比性边界](METHODOLOGY.md)
- [英文版入口](README.en.md)

## 发布范围

本仓库只包含脱敏的实验叙述、聚合指标和方法说明，不包含：

- 模型权重、量化文件或 Python 环境；
- OMP overlay、服务脚本或任何可执行代码；
- 原始 session JSONL、终端日志、工作区快照或数据集副本；
- API 密钥、令牌、个人路径或可识别信息。

## 日用策略

| 使用场景 | 路由 |
| --- | --- |
| 默认交互与自然 Vibe | 全 DeepSeek Flash |
| 本地模型试验或离线执行 | 显式启动本地 Qwen 服务 |
| 高可靠复杂推理 | 显式使用 Codex |

本地模型没有被删除。它保留用于离线任务、后端实验和需要本地 worker 的特定工作流；只是不会再成为默认路径。

## 许可

文档以 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 发布。
