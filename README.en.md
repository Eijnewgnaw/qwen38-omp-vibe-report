# Qwen3.8-27B + OMP Vibe: a local coding-agent deployment diary

This is a sanitized, experience-focused record of retaining a local Qwen3.8-27B 4-bit stack on WSL2 + RTX 3090 while evaluating remote and hybrid OMP Vibe routing for daily coding work.

## Conclusion

The current default is **All-DeepSeek Flash with natural Vibe**. The local Qwen service remains available as an explicit, on-demand capability rather than the default orchestration path.

On the same Matplotlib `Axes.clear()` repair task, natural All-DeepSeek Vibe completed with independent verification in 344.4 seconds. The natural DeepSeek-director + local-Qwen-worker run took 1495.8 seconds. It did not create useful parallelism and instead added local inference waits and a longer work chain.

This is not a universal ranking of models, tasks, or hardware. It shows that hybrid routing may not pay off when the director can execute well, the task has dependencies, and local workers do not shorten the critical path.

## Documents

- [Experiment report](REPORT.en.md)
- [Log and inference analysis: Hybrid versus All-DeepSeek Flash](HYBRID_VS_ALL_DSF_ANALYSIS.en.md)
- [Local Qwen deployment, framework, concurrency, and context study](LOCAL_QWEN_DEPLOYMENT.en.md)
- [Methodology and comparability limits](METHODOLOGY.en.md)
- [Chinese entry point](README.md)

## Publication scope

This repository contains only sanitized narrative, aggregate metrics, and methodology. It does not include model weights, quantizations, Python environments, OMP overlays, service scripts, executable code, raw session JSONL, terminal logs, workspaces, datasets, credentials, personal paths, or identifying information.

## Daily policy

| Scenario | Route |
| --- | --- |
| Default interaction and natural Vibe | All-DeepSeek Flash |
| Local experiments or offline execution | Explicitly start local Qwen |
| Highest-stakes complex reasoning | Explicitly use Codex |

The local model was not deleted. It remains available for offline work, backend research, and known-parallel batch subtasks; it is simply no longer the default route.

## License

Documentation is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
