# Log and inference analysis: Hybrid versus All-DeepSeek Flash

For the Chinese version, see [HYBRID_VS_ALL_DSF_ANALYSIS.md](HYBRID_VS_ALL_DSF_ANALYSIS.md).

## Executive conclusion

On the same Matplotlib `Axes.clear()` repair, Hybrid took 1495.8 seconds and All-DeepSeek Flash took 344.4 seconds. The logs do not support reducing the difference to “Qwen cannot code” or “Vibe concurrency failed.” The evidence supports a compound explanation:

1. Every long-input, short-output local Qwen tool turn was expensive.
2. Hybrid generated 45 local-model calls; the All-DeepSeek worker used 12.
3. Prefix caching was disabled for local vLLM, while DeepSeek reported substantial cache reads.
4. Qwen made structured-delivery and parallel-result association errors, creating avoidable follow-up work.
5. The Hybrid director created several serial work stages; All-DeepSeek used one cohesive implementation loop.
6. The repair did not contain two independent change branches, so concurrency could not offset local-call overhead.

These were single natural-Vibe runs at temperature 1.0. They explain the observed executions but do not rank the models' overall coding ability.

## Comparability

Both runs used the same user-task SHA-256, DeepSeek Flash high as the root director, the same repository base, tools, temperature, compaction threshold, and maximum concurrency of two. Delegation and worker selection remained natural.

The worker tiers were not fully equivalent. Hybrid mapped both fast and good to local Qwen with thinking disabled; only their task contracts differed. All-DeepSeek mapped fast to DeepSeek low and good to DeepSeek high. The Hybrid run used only fast/sonic workers, while All-DeepSeek used one good/task worker.

The comparison therefore includes worker model, reasoning tier, role description, and stochastic scheduling—not a single backend variable.

## End-to-end and model-call decomposition

| Metric | Hybrid | All-DeepSeek Flash |
| --- | ---: | ---: |
| End-to-end wall time | 1495.8 s | 344.4 s |
| Director calls | 34 | 32 |
| Worker calls | 45 | 12 |
| Total model calls | 79 | 44 |
| Cumulative director model duration | 310.9 s | 187.9 s |
| Cumulative worker model duration | 1013.1 s | 25.0 s |
| Total cumulative model duration | 1324.0 s | 212.9 s |
| Tool calls across the session tree | 111 | 47 |
| Effective worker overlap | none | none |

The wall-time gap was 1151.4 seconds. The difference in cumulative model duration was approximately 1111 seconds, or 96.5% of that gap. Cumulative model duration is not a strict critical-path metric, but the workers were effectively serial in these runs, so it clearly locates the dominant cost in model requests rather than test execution.

The relevant pytest commands usually took fractions of a second to about two seconds and cannot explain a twenty-minute difference.

## Inference mechanics: why short tool turns were still slow

A tool-using agent normally creates a separate model request after every tool result:

```text
model emits tool call
→ tool executes
→ OMP appends history and the new tool result
→ next model request begins
```

A persistent worker preserves session identity, history, and workspace. It does not guarantee that the previous request's complete KV state remains resident on the GPU indefinitely.

The first request in each new Qwen worker already contained about 14K tokens from system instructions, tool schemas, Vibe context, and the task. As tool results accumulated, the locate worker grew from roughly 14K to 26.8K input tokens per call. Even when output was only tens or hundreds of tokens, the model still had to prefill that long input.

Prefix caching was disabled in the stable local vLLM configuration, so each usage record included the full prompt. DeepSeek reported reused prefixes separately as cache reads:

| Worker | Uncached input | Cache read | Logical prompt volume |
| --- | ---: | ---: | ---: |
| Qwen | 794,675 | 0 | 794,675 |
| DeepSeek | 17,764 | 178,944 | 196,708 |

Qwen saw about four times the logical prompt volume because it made 45 calls rather than 12. DeepSeek's cache accounting also indicates server-side reuse of old prefixes. Cache semantics differ by provider, so cache reads should not be treated as literally free computation, but they are more favorable than complete repeated prefill.

## Hybrid's four serial work stages

| Qwen session | Calls | Cumulative model duration | Mean per call |
| --- | ---: | ---: | ---: |
| Locate implementation | 19 | 423.7 s | 22.3 s |
| Apply fix | 10 | 218.3 s | 21.8 s |
| Independent verification | 3 | 50.1 s | 16.7 s |
| Diagnose test collection | 13 | 320.9 s | 24.7 s |

The sessions followed a dependency chain with no overlap: locate, implement, verify, then diagnose. A maximum concurrency of two is only a capacity ceiling; it does not manufacture independent work.

The repair had one core change point, and locate–modify–verify is inherently ordered. Lack of overlap was not itself a scheduling failure. It meant that Hybrid gained no parallel benefit to offset its higher per-turn cost.

## Directly observed protocol overhead

### Structured `yield` failures

The implementation worker submitted an invalid `yield` structure twice before succeeding. The diagnostic worker did the same. The four extra model calls consumed approximately 141 seconds. The tool response explicitly requested an object containing `data` or `error`, so this overhead came from unreliable compliance with the worker-delivery protocol rather than code execution.

### `hub` communication loop

The locate worker used `hub` nine times, including a missing destination, a skipped send due to a queued message, repeated acknowledgements, and one wait of about two minutes. The worker did locate the code, but the communication pattern expanded both context and delay.

### Parallel test-result association error

The implementation worker launched two pytest commands in parallel. Raw tool-call identifiers show the correct association:

```text
test_axes.py   → 6 passed, 834 deselected
test_artist.py → 1 passed, 28 deselected
```

The worker's structured report reversed those results, and the verification worker repeated the same association error. The director therefore believed that `test_axes.py` had collected only 29 tests and created a diagnostic worker.

The diagnostic session proved that the environment was normal. This avoidable tail consumed 13 Qwen calls and 320.9 seconds of model duration.

## Why the director strategies diverged

The Hybrid director encountered large-file read difficulty and delegated symbol location to a fast worker after about 130 seconds. It then reused that worker and created separate implementation, verification, and diagnostic sessions.

The All-DeepSeek director explored for about 269 seconds before delegating, so its initial investigation was not faster. By delegation time, however, it had formed a substantially complete repair plan. It gave reproduction, editing, regression testing, and diff inspection to one DeepSeek-high good worker, which completed the cohesive loop in about 33 seconds.

The evidence therefore does not prove that the All-DeepSeek director is inherently a better planner. It shows that Hybrid delegated an open-ended location problem earlier, while All-DeepSeek delegated a converged implementation package later. The latter task shape required fewer handoffs and model turns.

Both runs used temperature 1.0 and were not repeated, so stochastic sampling may also have contributed to the strategy split.

## Explanations supported and unsupported by the logs

The logs rule out GPU OOM, contention between two concurrent Qwen workers, slow Python tests, and total inability of Qwen to implement the fix. The implementation worker did change the code, reproduce the behavior, and run tests.

The logs do not prove that All-DeepSeek is always 4.34 times faster, that enabling Qwen thinking would necessarily help, that prefix caching would safely solve the problem, or that forced concurrency would improve a one-file repair.

## Final assessment

Hybrid lost on system efficiency for high-frequency tool loops, not on the bare ability to write a patch. Local Qwen paid a high cost for each 14K–27K-history turn, while the workflow generated 45 calls, structured-protocol retries, and one unnecessary diagnostic branch caused by swapped test-result attribution. All-DeepSeek combined faster remote serving, reported prefix reuse, and a cohesive task package to finish with 12 worker calls.

All-DeepSeek is therefore the current daily default for interaction efficiency. Local Qwen remains better suited to bounded packages that can be specified and delivered in one turn, genuinely independent parallel tasks, and work with explicit privacy or offline requirements.

Future Hybrid work should prioritize prefix reuse, reliable `yield` formatting, correct parallel-tool result association, bounded communication, and cohesive delegation before increasing worker count.

## Data scope

This report publishes aggregate metrics and behavioral evidence only. Raw parent/child sessions, tool-call identifiers, terminal output, service configurations, and workspace snapshots remain local.
