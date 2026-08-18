# Report: Should local Qwen workers be the default for OMP Vibe?

For the Chinese version, see [REPORT.md](REPORT.md). For the local serving-stack, concurrency, and context experiments, see [LOCAL_QWEN_DEPLOYMENT.md](LOCAL_QWEN_DEPLOYMENT.md).

## Executive summary

The original goal was to keep Qwen3.8-27B 4-bit running locally on WSL2 + RTX 3090 while using a stronger remote model as an OMP Vibe director. The director would delegate execution-oriented, parallel subtasks to local Qwen workers, reducing remote usage while preserving a natural agent workflow.

The results split into two categories:

- In a deliberately forced two-worker, two-bug fixture, All-Qwen, Hybrid, and All-Codex all passed 5/5 tests. Hybrid was faster than All-Qwen but slower than All-Codex.
- In a more realistic natural-Vibe Matplotlib repair, All-DeepSeek Flash was faster. The DeepSeek-director + local-Qwen-worker path did not form useful parallelism and created a longer work chain.

The current daily default is therefore **All-DeepSeek Flash with natural Vibe**. Local Qwen inference is retained as an explicit, on-demand capability for offline work, backend research, and tasks known to divide cleanly.

## Deployment snapshot

The local baseline was a text-only Qwen3.8-27B AWQ W4A16 group-128 model served by vLLM 0.27.1 on one RTX 3090. The frozen stable profile used FP8 KV cache, a 49,152-token API window, two sequences, natural EOS with an 8,192-token output cap, MTP off, and prefix cache off. On WSL, the V1 runner was retained because the newer runner had a UVA compatibility issue in this setup.

The API window and the stable working budget are intentionally distinct: about 32K tokens per worker were repeatedly validated as input/history budget. The experiment does not claim that two 49K slots can both be filled.

## Forced two-worker fixture: routing and accounting validation

The same small Python fixture was explicitly split into two independent fixes. The prompt, workspace, tool permissions, and success criterion were held constant. This is a test of OMP routing, persistent worker sessions, overlap, and accounting—not a general coding benchmark.

| Route | Result | Wall time | Note |
| --- | --- | ---: | --- |
| All-Qwen (vLLM) | 5/5 tests | 528.5 s | two local workers |
| Hybrid (Codex director + Qwen workers) | 5/5 tests | 368.5 s | faster than All-Qwen |
| All-Codex | 5/5 tests | 87.8 s | fastest arm |

Relative to All-Codex, Hybrid reduced counted remote tokens by 47.97% and API-equivalent cost by 55.52%, while taking about 319.6% longer end-to-end. In plain terms: **Hybrid spent less remotely; All-Codex finished sooner.** Qwen and Codex tokens are not added together because their tokenizers differ.

## Natural Vibe: Matplotlib repair

The task was to repair `Axes.clear()`/`Axes.cla()` so clearing artists also removes their `axes` and `figure` references. It required repository inspection, a minimal patch, and relevant tests. The director was free to decide whether and how to delegate.

| Route | Task outcome | Wall time | Delegation observed |
| --- | --- | ---: | --- |
| DeepSeek director + Qwen workers | Session records show a repair and passing tests | 1495.8 s | four fast workers, largely serial; no good worker and no useful C2 overlap |
| All-DeepSeek Flash | Independently verified in a clean offline container | 344.4 s | one fast worker; also serial, but with a shorter chain |

The All-DeepSeek workspace was independently re-checked with an offline behavioral assertion, 16 related axes tests, and three related figure tests. The earlier Hybrid run preserved session and test evidence but not an equally reproducible final workspace snapshot. It is used for workflow and latency observations, not as equally strong patch-quality evidence.

## Why the hybrid path did not pay off

1. The repair behaved like a dependency chain—locate, change, verify—rather than two independent chunks of work.
2. Natural Vibe does not create parallel work simply because fast and good roles exist in a configuration.
3. On a single GPU, local Qwen adds first-token and inference waits. If workers do not reduce the critical path, those waits appear directly in end-to-end time.
4. Extra progress, timeout, and takeover controls can make dense local workers more predictable, but they also alter natural Vibe behavior and add operational complexity.

## Decision

- **Default `omp`:** All-DeepSeek Flash + natural Vibe, with no forced worker count, task split, or runtime progress gate.
- **Local Qwen:** preserved and available through an explicit local/hybrid path; suitable for offline/private work and bounded, truly independent subtasks.
- **Codex:** an explicit choice for high-reliability complex reasoning.

The choice optimizes for a fast, reliable coding-agent interaction rather than using local GPU capacity by default.

## Limitations

- The sample is small and is not a leaderboard or universal model ranking.
- Natural Vibe scheduling is itself an experimental variable, so it is not a pure model A/B test.
- Provider tokenization, cache semantics, and API-equivalent pricing differ.
- Results are sensitive to the RTX 3090, WSL2, quantization, versions, and local runner configuration.
- The public repository intentionally omits raw sessions, containers, and source snapshots.

## Practical lessons

1. Validate routing and accounting with a tiny forced fixture before using natural agent tasks.
2. Seek concurrency only for independently deliverable subtasks; a strong serial director often wins on dependency chains.
3. Separate API window from stable context budget.
4. Prefer on-demand model loading and idle unload on a personal single-GPU machine.
5. Evaluate verifier success, TTFT, wall time, tool behavior, and VRAM headroom together—not token throughput alone.
