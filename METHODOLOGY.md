# 方法与可比性边界

English version: [METHODOLOGY.en.md](METHODOLOGY.en.md).

## 环境范围

- 宿主：Windows + WSL2。
- GPU：单张 RTX 3090（24 GB）。
- 本地模型：Qwen3.8-27B text-only 4-bit；实验时由 vLLM 提供 OpenAI-compatible 服务。
- 编排：OMP Vibe 持久 worker 会话。
- 远端模型：DeepSeek Flash、Codex Sol；成本字段是按提供方 token 计数得到的 API 等价估算，不代表订阅实际扣费。

## 如何记录

每一轮实验保留私有原始资料，并从父/子 session 记录提取：

- 实际 provider/model 路由、调用数和角色；
- input、output、cache read/write、reasoning token（仅在提供时记录）；
- wall time、单调用 duration、TTFT；
- 工具调用、worker 创建、并行时间重叠、重试与超时；
- GPU 显存与服务状态；
- 最终 diff 与独立测试结果。

公开报告只使用聚合后的数字与最小必要的 patch 行为描述。

## 两类实验不能混为一谈

1. **强制双 worker 小型 fixture**：提示词明确分配两个独立 bug，主要测量模型路由、并发、会话和统计链路。
2. **自然 Vibe 代码任务**：Director 自行决定是否委派、委派给谁以及何时合并，更接近日用体验，但更容易被任务依赖、模型自主策略和测试选择影响。

自然 Vibe 的结果不应被解释成严格的单变量模型能力对照；只有相同任务、相同初始工作区、相同提示、相同 verifier、相同资源限制的一组运行，才可作有限的 A/B 比较。

## Matplotlib 人工复现实验

任务是修复 `Axes.clear()`/`Axes.cla()` 在清空 artist 时未解除 `axes` 与 `figure` 引用的问题。两次自然 Vibe 均使用同一问题描述、相同容器基础、禁用网络和相同验证命令；任务允许 Agent 选择自己的分工。

All-DeepSeek 的最终工作区还由独立、干净的无网络容器复核：行为断言通过；相关 axes 测试为 16 passed；相关 figure 测试为 3 passed。

Hybrid 的较早一轮保存了会话与测试证据，但没有保留同等粒度的最终工作区快照。因此，报告把它用于工作流与时延观察，而不把它作为同等级的 patch-quality 证据。
