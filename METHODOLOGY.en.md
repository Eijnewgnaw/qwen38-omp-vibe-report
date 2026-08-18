# Methodology and comparability limits

For the Chinese version, see [METHODOLOGY.md](METHODOLOGY.md).

## Scope

- Host: Windows + WSL2.
- GPU: one RTX 3090 (24 GB).
- Local model: Qwen3.8-27B text-only 4-bit, served through an OpenAI-compatible endpoint.
- Orchestration: OMP Vibe persistent worker sessions.
- Remote models: DeepSeek Flash and Codex Sol. Dollar figures are API-equivalent estimates calculated from provider token fields, not actual subscription charges.

## What was recorded privately

Each run retained raw private evidence and extracted the following from parent and child sessions when available:

- actual provider/model routing, call count, and role;
- input, output, cache-read/write, and reasoning tokens;
- wall time, per-call duration, and TTFT;
- tool calls, worker creation, parallel overlap, retries, and timeouts;
- GPU memory and service state;
- final diff and independent test outcome.

The public repository contains only aggregate values and the minimum patch-behavior description necessary to explain the result.

## Do not merge these experiment classes

1. **Forced two-worker fixture:** the prompt assigns two independent bugs. It tests routing, concurrency, session handling, and accounting.
2. **Natural-Vibe coding task:** the director is free to decide whether and how to delegate. It better reflects daily use but is more affected by task dependencies and agent policy.

Natural-Vibe results should not be interpreted as a strict one-variable model benchmark. Only paired runs with the same task, initial workspace, prompt, verifier, and resource limits support a narrow A/B claim.

## Matplotlib replication note

The manual task repaired `Axes.clear()`/`Axes.cla()` so that clearing artists removes stale `axes` and `figure` references. Both natural-Vibe trials used the same problem description, container base, disabled network, and verification command, while allowing the agents to choose their own work division.

The All-DeepSeek final workspace was independently checked in a clean offline container: the behavioral assertion passed, 16 related axes tests passed, and three related figure tests passed. The earlier Hybrid run does not have a final workspace snapshot of equivalent granularity, which limits patch-quality comparison.
