# 实验报告：本地 Qwen worker 值不值得成为 OMP Vibe 默认？

英文版见 [REPORT.en.md](REPORT.en.md)。本地部署前的框架比较，以及 Qwen 的上下文、KV cache 与并发实验，见中英双语附录：[LOCAL_QWEN_DEPLOYMENT.md](LOCAL_QWEN_DEPLOYMENT.md)。

## 执行摘要

这套部署一开始的目标是：让远端模型负责 Director，本地 Qwen3.8-27B 4-bit 承担并行的执行型 worker，以减少远端 token 与成本，并尽量保留 OMP Vibe 的自然协作体验。

实际结果分成两部分：

- 在一个**明确强制双 worker**的微型双 bug 修复 fixture 中，三种路由都完成 5/5 测试；Hybrid 比全 Qwen 更快，但比全 Codex 慢。
- 在更接近真实使用的**自然 Vibe** Matplotlib 修复任务中，纯 DeepSeek Flash 更快，而且自然委派没有形成有效的本地并行。故当前默认切换到 All-DeepSeek；本地 Qwen 保留为按需能力。

这不是“本地模型无用”的结论，而是“本机单卡、27B 4-bit、任务依赖明显、Director 可自行执行”的组合下，默认混合编排的收益不足以抵消复杂度。

## 部署快照

本地服务基线为 WSL2 上的单 RTX 3090（24 GB）、Qwen3.8-27B text-only 4-bit、vLLM OpenAI-compatible API。已验证的本地日用参数为并发 2、约 49K 每请求上下文窗口、8K 最大生成上限；模型以按需服务形式保留，不再默认常驻占用显存。

社区 long/MTP 变体曾有更高短吞吐，但在 Agent 工作流中没有证明出足以抵消稳定性与运维复杂度的收益。因此保留官方稳定服务栈作为本地能力的基础。

## 强制双 worker：路由与统计链路验证

同一个小型 Python 双 bug fixture 被明确拆为两个独立修改，使用相同工具权限、并发参数和成功条件。该测试不代表通用代码能力，主要验证 OMP 可以创建并收集两个持久 worker 的结果。

| 路由 | 结果 | wall time | 备注 |
| --- | --- | ---: | --- |
| 全 Qwen（vLLM） | 5/5 tests | 528.5 s | 两个本地 worker |
| Hybrid（Codex Director + Qwen workers） | 5/5 tests | 368.5 s | 相比全 Qwen 更快 |
| 全 Codex | 5/5 tests | 87.8 s | 本组最快 |

Hybrid 相比全 Codex，记录的远端 token 减少 47.97%，API 等价成本减少 55.52%，但端到端时间增加约 319.6%。这句话的含义是：**Hybrid 更省远端用量与等价成本；All-Codex 更省时间。** 不应把不同 tokenizer 的 Qwen token 与 Codex token 直接相加。

## 自然 Vibe：Matplotlib 代码修复

任务为修复 `Axes.clear()`/`Axes.cla()` 对已从 axes 移除 artist 的引用清理。它要求探索源码、实现一个最小 patch，并运行相关测试；没有要求 Director 必须创建固定数量的 worker。

| 路由 | 任务结果 | wall time | 自然委派现象 |
| --- | --- | ---: | --- |
| DeepSeek Director + Qwen workers | 会话记录显示修复与测试通过 | 1495.8 s | 创建 4 个 fast worker，基本串行；无 good worker、无有效 C2 overlap |
| All-DeepSeek Flash | 独立无网络复核通过 | 344.4 s | 创建 1 个 fast worker；仍是串行，但工作链短 |

All-DeepSeek 的独立复核包括一个引用解除行为断言、16 个相关 axes 测试和 3 个相关 figure 测试。Hybrid 那一轮缺少可等价复核的最终工作区快照，故不将两者的 patch 质量声称为严格同等级；时延与委派模式仍有参考价值。

## 为什么 Hybrid 没有获益

1. 任务更像“先定位、再修改、再验证”的依赖链，而不是两块可以独立并行的工作。
2. 自然 Vibe 会根据任务自行决定角色；它不会因为配置中有 fast/good worker 就强行凑出并发。
3. 本地 Qwen 在单卡上有较长首 token 和推理等待。worker 没有并行缩短关键路径时，这些等待会直接叠加到总时间。
4. 为让 dense 本地 worker 更可靠而增加的超时、接管和进度护栏，虽然能改善可控性，也会改变自然 Vibe 的行为，并增加运维复杂度。

## 当前决策

- 默认 `omp`：All-DeepSeek Flash + 自然 Vibe，不添加强制拆分、固定 worker 数或运行时进度护栏。
- 本地 Qwen：保留安装、模型、后端和显式启动入口；用于离线、私有代码、后端研究或已知可并行的批量子任务。
- Codex：保留为需要高可靠复杂推理时的显式选择。

这个默认选择偏向“交互像一个快速可靠的远端 coding agent”，而不是为了使用本地 GPU 而让每个任务都经过本地 worker。

## 局限性

- 样本量很小，不能外推为广泛 benchmark 排名。
- 自然 Vibe 的调度策略本身是实验对象的一部分，因此不是纯模型 A/B。
- 不同 provider 的 tokenization、缓存语义和 API 等价价格不相同。
- 本地结果对 RTX 3090、WSL2、量化方式和模型版本敏感。
- 公开版不附原始 session、容器或完整代码，重现需要自行准备相同环境。

## 可复用的经验

1. 先用强制小 fixture 验证路由、会话和统计是否正确，再用真实任务观察自然调度。
2. 只在子任务真正独立、可预期交付时追求并发；存在依赖链时，串行的强 Director 往往更好。
3. 将本地服务做成按需加载并设置空闲卸载，比默认长时间占满显存更适合个人单卡使用。
4. 同时报端到端 wall time、独立 verifier 结果和各 provider 的 token/成本；只看 tok/s 或 token 节省都会误导决策。
