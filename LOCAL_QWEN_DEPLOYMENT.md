# 本地 Qwen3.8-27B 部署：框架、并发与上下文探索 / Local Qwen3.8-27B deployment: frameworks, concurrency, and context

## 1. 目标与硬件 / Goal and hardware

**中文。** 目标不是把单张 24 GB 显卡变成多用户集群，而是在 WSL2 + RTX 3090 上运行 Qwen3.8-27B text-only 4-bit，并为 OMP Vibe 保留两个可同时工作的本地 worker。重点是：服务能按需装载、双并发不会 OOM、长历史能保留、工具调用正确，以及闲置时释放显存。

**English.** The goal was not to turn one 24 GB GPU into a multi-user cluster. It was to serve Qwen3.8-27B text-only 4-bit on WSL2 + RTX 3090, retaining two simultaneously usable local workers for OMP Vibe. The priorities were on-demand loading, no OOM at dual concurrency, usable long history, correct tool calls, and VRAM release while idle.

## 2. 部署前的框架选择 / Framework choices before deployment

| 框架 / framework | 角色与观察 / role and observation | 本轮定位 / decision in this work |
| --- | --- | --- |
| **vLLM** | AWQ/Compressed-Tensors 直接服务；连续批处理、调度与 OpenAI-compatible API 最适合两个并行 worker。WSL 上固定 V1 runner（`VLLM_USE_V2_MODEL_RUNNER=0`），以规避当时的 UVA/V2 兼容问题。 | 本地高性能 API 基线；保留为显式本地服务。 |
| **llama.cpp** | GGUF 路线，KV 可用 Q8_0，资源占用较低，参数直接；但本卡上的长上下文双并发 TTFT/总时延较慢。 | 可靠备用后端、GGUF 验证路径。 |
| **LM Studio / llmster** | 可在 WSL headless 运行；负责模型导入、runtime 选择、加载参数和 OpenAI API。底层 runtime 仍为 llama.cpp，并非第三个推理引擎。 | GGUF 模型管理器与备用服务，不作默认双 worker 后端。 |
| **Ollama** | 适合简单拉取、交互和单机模型管理；本轮没有把它作为受控 C2 长上下文的正式性能基线。它的便利性不等于在本机 Qwen/OMP 双 worker 调度上的最佳结果。 | 保留为可选日常工具，不纳入本报告的数值比较。 |
| **llama-swap + systemd user service** | 只管理一个受监督的 vLLM 子进程，解决“模型长期占满显存”问题。 | 按需加载与 TTL 卸载的生命周期层，不是推理引擎。 |

**中文。** 最终不把 LM Studio 与 vLLM当作同一种产品比较：前者主要改善 GGUF 管理与易用性，后者主要改善 AWQ 服务调度。两者使用不同量化和 KV 方案，所以公开数值代表“可部署栈”而非严格的单引擎微基准。

**English.** LM Studio and vLLM were not treated as the same kind of product: the former primarily improves GGUF management and ease of use; the latter primarily improves AWQ serving and scheduling. They use different quantization and KV schemes, so the public numbers compare deployable stacks rather than a pure single-engine microbenchmark.

## 3. 最终稳定的本地 vLLM 基线 / Frozen local vLLM baseline

| 项目 / item | 值 / value |
| --- | --- |
| Model | Qwen3.8-27B, text-only, AWQ W4A16 group 128 |
| Server | vLLM 0.27.1, OpenAI-compatible API, loopback-only |
| KV cache | FP8 |
| Interface window | 49,152 tokens per request |
| Stable input/history budget | 32K tokens per worker |
| Output cap | 8,192 tokens, natural EOS |
| Local concurrency | 2 (`max-num-seqs=2`; OMP local in-flight cap 2) |
| MTP / prefix cache | Off / off for the stable baseline |
| WSL runner | V1 (`VLLM_USE_V2_MODEL_RUNNER=0`) |

**中文。** 这里的 49,152 是 API 接口窗口；它不等于“两个会话都能塞满 49,152 token”。32K 是反复验证过的输入/历史工作预算，8K 是自然输出和工具回灌余量。实际 GPU KV 容量在不同启动检查中为 89,877–101,112 token，因此足以覆盖已验证的双 32K 自然请求，但不应用它宣传双满 49K。

**English.** The 49,152 figure is the API-facing request window; it does not mean that two sessions can both be filled to 49,152 tokens. 32K is the repeatedly validated input/history working budget, with 8K reserved for natural output and tool round-trips. Observed GPU KV capacity varied from 89,877 to 101,112 tokens across startup checks, which covers the validated dual-32K natural workload but does not justify claiming two fully populated 49K slots.

## 4. 上下文和并发实验 / Context and concurrency experiments

### 4.1 双 32K 自然 Agent 形状 / Dual 32K natural-agent shape

两个请求同时提交；每路约 32K prompt，`max_tokens=8192`，允许自然 EOS，而不是强制模型输出 8K。passphrase/retention 与 HTTP 错误被同时记录。

Two requests were submitted together, each with roughly a 32K prompt and `max_tokens=8192`. Natural EOS was allowed rather than forcing an 8K completion. Passphrase retention and HTTP errors were recorded.

| 后端 / backend | Repetitions | Success | Mean wall time | Mean TTFT | Peak VRAM | Interpretation |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| Official vLLM AWQ + FP8 KV | 2 | 4/4 | 57.482 s | 42.199 s | formal sampled run | Fastest validated local C2 stack in this comparison. |
| Native llama.cpp GGUF + Q8_0 KV | 2 | 4/4 | 94.668 s | 72.786 s | formal sampled run | Works, but markedly slower under long C2. |
| LM Studio/llmster llama.cpp + Q8_0 KV | 1 | 2/2 | 80.227 s | 52.471 s | 20,796 MiB | Actual dual-slot parallelism, but one repetition only. |

**中文。** LM Studio 两个请求各自约 80 秒完成，而总 wall time 也是 80.227 秒，说明不是串行；但它相对 vLLM 的平均 wall time 慢 39.57%。LM Studio 相比原生 llama.cpp 的样本均值快 15.25%，但只有一次重复，不能把这个差异当成稳定结论。

**English.** Both LM Studio requests individually took about 80 seconds while total wall time was 80.227 seconds, demonstrating real dual-slot parallelism rather than serial execution. Still, it was 39.57% slower than the vLLM mean. It was 15.25% faster than the native llama.cpp sample mean, but that is only one repetition and is not a stability claim.

### 4.2 社区 MTP 配置的扩大并发探索 / Community MTP profiles at wider concurrency

社区候选被放在独立环境中测试，不覆盖稳定基线。短提示 aggregate output throughput 如下：

Community candidates were tested in an isolated environment without replacing the stable baseline. Short-prompt aggregate output throughput was:

| Profile | C1 | C2 | C4 | C8 |
| --- | ---: | ---: | ---: | ---: |
| Community long, FP8 KV, 150K, MTP-3 | 67.18 tok/s | 117.26 tok/s | 219.61 tok/s | 330.01 tok/s |
| Community fast, BF16 KV, 65,536, MTP-4 | 72.04 tok/s | 131.62 tok/s | 201.40 tok/s | 281.63 tok/s |
| Official stable control, C2 | — | 82.54 tok/s | — | — |

在自然 EOS、较长 prompt 的工作负载中，社区 fast 的 C2 吞吐比官方 control 高 59.5%。但同一个 OMP Vibe 双 bug fixture 的三次配对运行中，中位 wall time 仅从 368.477 s 降到 336.732 s（8.62%），低于预设 15% 替换门槛；Qwen TTFT 中位数反而上升 6.82%，工具错误多 2 个。故社区 profile 被保留为实验路径，而非默认日用配置。

On natural-EOS, longer-prompt workloads, community fast delivered 59.5% higher C2 throughput than the official control. Yet in three paired OMP Vibe double-bug runs, median wall time improved only from 368.477 s to 336.732 s (8.62%), below the pre-set 15% replacement threshold. Median Qwen TTFT was 6.82% higher and there were two more tool-error results. The community profiles therefore remain experimental rather than the daily default.

### 4.3 为什么最终是并发 2 / Why concurrency 2 won

**中文。** C4/C8 可以获得更高 aggregate throughput，社区 C4 的自然 Agent-shaped 负载相对 C2 增益约 23.6%；但平均 TTFT 从约 20.79 s 升到约 34.48 s（约 +65.8%）。个人日常主任务更关心两个独立 worker 的可交互性与显存余量，所以最终选择两个并行槽，而不是把吞吐最大化。

**English.** C4/C8 achieved higher aggregate throughput; community C4 gained about 23.6% over C2 for an agent-shaped natural workload. However, mean TTFT rose from about 20.79 s to about 34.48 s (about +65.8%). A personal primary task benefits more from two responsive independent workers and VRAM headroom than from maximum aggregate throughput, so two parallel slots were selected.

## 5. 显存生命周期 / VRAM lifecycle

**中文。** 本地模型不应因打开 OMP 就永久占用约 22–24 GiB 显存。服务层采用“请求触发完整加载、最后一个会话空闲后 TTL 卸载”的策略；正式 TTL 为 15 分钟。实测冷启动中位数约 46.455 s，p95 约 51.109 s；卸载后显存回到约 745 MiB 的测试态基线。活请求期间安全 unload 被拒绝，以避免中断生成。

**English.** Opening OMP should not permanently reserve roughly 22–24 GiB of VRAM. The service layer uses full load-on-request and TTL unload after the last session becomes idle; the production TTL was 15 minutes. Measured cold-start median was about 46.455 s and p95 about 51.109 s; after unload, VRAM returned to an approximately 745 MiB test-state baseline. Safe unload is rejected during active requests to avoid interrupting generation.

## 6. 可复用结论 / Reusable conclusions

1. **先选量化/服务格式，再选 UI。** AWQ + vLLM 适合本机并行服务；GGUF + llama.cpp/LM Studio 适合管理、验证和较低资源压力。
2. **把接口窗口和稳定预算分开。** 49K endpoint is not a promise of two fully filled 49K contexts.
3. **并发要按响应性决策。** C4/C8 的吞吐更高不表示对单个 Vibe 主任务更好；C2 是本机的平衡点。
4. **按需模型管理很重要。** 让模型 TTL 卸载保留桌面显存，实际体验比让 27B 服务永久常驻更好。
5. **不要只看 tok/s。** 同时看成功率、TTFT、端到端 wall time、工具调用和显存余量。
