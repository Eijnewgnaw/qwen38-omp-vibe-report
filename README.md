# Qwen3.8-27B + OMP Vibe: a local coding-agent deployment diary

中文在前，English follows. This is a sanitized, experience-focused record of keeping a local Qwen3.8-27B 4-bit stack on WSL2 + RTX 3090 while evaluating remote and hybrid OMP Vibe routing for daily coding work.

## 中文摘要

当前默认采用 **All-DeepSeek Flash + 自然 Vibe**；本地 Qwen 服务保留为显式按需能力，而不再是默认编排路径。

在同一个 Matplotlib `Axes.clear()` 修复任务上，纯 DeepSeek 自然 Vibe 完成并通过独立复核，端到端用时 344.4 秒；DeepSeek Director + 本地 Qwen worker 的自然 Vibe 记录用时 1495.8 秒。后者没有形成有效并行，反而引入本地推理等待与更长的工作链。

这不是对所有模型、任务或硬件的普适排名。它说明：当 Director 已有足够执行能力、任务存在依赖链、而本地 worker 未能缩短关键路径时，混合路由未必有实际收益。

- [中文主报告](REPORT.md)
- [English report](REPORT.en.md)
- [本地 Qwen 部署、框架与上下文/并发探索（中英双语）](LOCAL_QWEN_DEPLOYMENT.md)
- [中文方法说明](METHODOLOGY.md)
- [Methodology in English](METHODOLOGY.en.md)

## English summary

The current default is **All-DeepSeek Flash with natural Vibe**. The local Qwen service is retained as an explicit, on-demand capability rather than the default orchestration path.

On the same Matplotlib `Axes.clear()` repair task, natural All-DeepSeek Vibe completed with independent verification in 344.4 seconds. The natural DeepSeek-director + local-Qwen-worker run recorded 1495.8 seconds. It did not create useful parallelism and instead added local inference waits and a longer work chain.

This is not a universal ranking of models, tasks, or hardware. It shows that hybrid routing may not pay off when the director can execute well, the task has dependencies, and local workers do not shorten the critical path.

## Publication scope / 发布范围

This repository contains only sanitized narrative, aggregate metrics, and methodology. It does **not** include:

- model weights, quantizations, or Python environments;
- OMP overlays, service scripts, or executable code;
- raw session JSONL, terminal logs, workspaces, or dataset copies;
- API keys, tokens, personal paths, or identifying information.

## Daily policy / 日用策略

| Scenario / 场景 | Route / 路由 |
| --- | --- |
| Default interaction and natural Vibe / 默认交互与自然 Vibe | All-DeepSeek Flash |
| Local experiments or offline execution / 本地试验或离线执行 | Explicitly start local Qwen |
| Highest-stakes complex reasoning / 高可靠复杂推理 | Explicitly use Codex |

The local model was not deleted. It remains available for offline work, backend research, and known-parallel batch subtasks; it is simply no longer the default route.

## License / 许可

Documentation is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
