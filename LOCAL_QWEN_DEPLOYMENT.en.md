# Local Qwen3.8-27B deployment: frameworks, concurrency, and context

For the Chinese version, see [LOCAL_QWEN_DEPLOYMENT.md](LOCAL_QWEN_DEPLOYMENT.md).

## Goal and hardware

The aim was not to turn one 24 GB GPU into a multi-user cluster. It was to run Qwen3.8-27B text-only 4-bit on WSL2 + RTX 3090 while retaining two simultaneous local OMP Vibe workers. The priorities were on-demand loading, no OOM at dual concurrency, usable long history, correct tool calls, and VRAM release while idle.

## Framework choices before deployment

| Framework | Role and observation | Decision in this work |
| --- | --- | --- |
| vLLM | Serves AWQ/Compressed-Tensors directly; continuous batching, scheduling, and an OpenAI-compatible API suit two parallel workers. The WSL deployment was fixed to the V1 runner (`VLLM_USE_V2_MODEL_RUNNER=0`) because of a contemporary UVA/V2 compatibility issue. | Local high-performance API baseline; retained as an explicit local service. |
| llama.cpp | Uses the GGUF path, supports Q8_0 KV cache, has lower resource use and direct controls, but showed slower TTFT and total latency for long-context dual concurrency on this GPU. | Reliable fallback backend and GGUF validation path. |
| LM Studio / llmster | Runs headlessly in WSL and manages model import, runtime choice, load parameters, and an OpenAI-compatible API. Its runtime is still llama.cpp; it is not a third inference engine. | GGUF model manager and backup service, not the default two-worker backend. |
| Ollama | Convenient for simple pulls, interaction, and single-machine model management. It was not used as a formal controlled C2 long-context performance baseline. | Optional daily tool; excluded from the numerical comparison in this report. |
| llama-swap plus a systemd user service | Supervises one vLLM child process and addresses long-lived VRAM occupation. | On-demand loading and TTL-unload lifecycle layer, not an inference engine. |

LM Studio and vLLM should not be treated as the same kind of product: the former primarily improves GGUF management and usability, while the latter primarily improves AWQ serving and scheduling. They use different quantization and KV schemes, so the figures compare deployable stacks rather than a pure one-engine microbenchmark.

## Frozen stable local vLLM baseline

| Item | Value |
| --- | --- |
| Model | Qwen3.8-27B, text-only, AWQ W4A16 group 128 |
| Server | vLLM 0.27.1, OpenAI-compatible API, loopback-only |
| KV cache | FP8 |
| Interface window | 49,152 tokens per request |
| Stable input/history budget | 32K tokens per worker |
| Output cap | 8,192 tokens with natural EOS |
| Local concurrency | 2 (`max-num-seqs=2`; OMP local in-flight cap 2) |
| MTP / prefix cache | Off / off |
| WSL runner | V1 (`VLLM_USE_V2_MODEL_RUNNER=0`) |

The 49,152 figure is an API-facing request window; it does not mean that two sessions can both be filled to 49,152 tokens. The 32K budget was repeatedly validated for input/history, while 8K is reserved for natural output and tool round-trips. Observed GPU KV capacity ranged from 89,877 to 101,112 tokens across startup checks. That covers the validated dual-32K natural workload, but does not justify claiming two fully populated 49K slots.

## Context and concurrency experiments

### Dual-32K natural-agent shape

Two requests were submitted at the same time, each with roughly a 32K prompt and `max_tokens=8192`. Natural EOS was allowed instead of forcing an 8K completion. Passphrase retention and HTTP errors were recorded together.

| Backend | Repetitions | Success | Mean wall time | Mean TTFT | Peak VRAM | Interpretation |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| Official vLLM, AWQ + FP8 KV | 2 | 4/4 | 57.482 s | 42.199 s | formal sampled run | Fastest validated local C2 stack in this comparison. |
| Native llama.cpp, GGUF + Q8_0 KV | 2 | 4/4 | 94.668 s | 72.786 s | formal sampled run | Works, but is markedly slower under long C2. |
| LM Studio/llmster llama.cpp, Q8_0 KV | 1 | 2/2 | 80.227 s | 52.471 s | 20,796 MiB | Actual dual-slot parallelism, but only one repetition. |

Both LM Studio requests individually took about 80 seconds while the overall wall time was 80.227 seconds, demonstrating real dual-slot parallelism rather than serialization. It was nevertheless 39.57% slower than the vLLM mean. It was 15.25% faster than the native llama.cpp sample mean, but this came from one repetition and is not a stability claim.

### Wider-concurrency exploration with community MTP profiles

Community candidates ran in isolated environments and did not overwrite the stable baseline. Aggregate short-prompt output throughput was:

| Profile | C1 | C2 | C4 | C8 |
| --- | ---: | ---: | ---: | ---: |
| Community long, FP8 KV, 150K, MTP-3 | 67.18 tok/s | 117.26 tok/s | 219.61 tok/s | 330.01 tok/s |
| Community fast, BF16 KV, 65,536, MTP-4 | 72.04 tok/s | 131.62 tok/s | 201.40 tok/s | 281.63 tok/s |
| Official stable control, C2 | — | 82.54 tok/s | — | — |

For a natural-EOS, longer-prompt workload, community fast produced 59.5% higher C2 throughput than the official control. Yet across three paired runs of the same OMP Vibe double-bug fixture, median wall time moved only from 368.477 s to 336.732 s, an 8.62% gain below the pre-set 15% replacement threshold. Median Qwen TTFT was 6.82% higher and there were two additional tool-error results. The community profiles remain experimental rather than the default daily configuration.

### Why concurrency 2 was selected

C4/C8 achieved higher aggregate throughput. Community C4 improved by about 23.6% over C2 for an agent-shaped natural workload, but mean TTFT rose from roughly 20.79 s to 34.48 s, about 65.8%. A personal primary task benefits more from two responsive independent workers and VRAM headroom than from maximal aggregate throughput, so two parallel slots were selected.

## VRAM lifecycle

Opening OMP should not permanently reserve roughly 22–24 GiB of VRAM. The service layer uses full load-on-request and TTL unload after the last session becomes idle; the production TTL was 15 minutes. Measured cold-start median was about 46.455 s and p95 about 51.109 s. After unload, VRAM returned to an approximately 745 MiB test-state baseline. Safe unload is rejected during active requests to avoid interrupting generation.

## Reusable conclusions

1. Choose quantization and serving format before choosing a UI: AWQ + vLLM suits local parallel serving; GGUF + llama.cpp/LM Studio suits management, validation, and lower resource pressure.
2. Separate the interface window from the stable budget: a 49K endpoint is not a promise of two full 49K contexts.
3. Choose concurrency for responsiveness: higher C4/C8 throughput does not make a single Vibe primary task better; C2 is the local balance point.
4. On-demand model management matters: TTL unload preserves desktop VRAM and is more pleasant than keeping a 27B server resident forever.
5. Do not look only at tokens per second: consider verifier success, TTFT, end-to-end wall time, tool behavior, and VRAM headroom together.
