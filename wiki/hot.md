---
type: meta
title: Hot Cache
status: developing
created: 2026-10-18
updated: 2026-10-31
tags:
  - meta
  - hot-cache
---

# Recent Context

## Last Updated

2026-10-31: AgentDesk phase 3 (harness hardening ablation, approvals, idempotency, audit log) completed and saved.

## Key Recent Facts

- AgentDesk (`~/projects/agentdesk`): multi-tenant small-model support-agent platform on vLLM; see [[AgentDesk]]. Phase 1 commit `6e7132a`.
- Laptop GPU: RTX 3050 6 GB works with native Docker Engine (`docker context default`), not Docker Desktop.
- Qwen3-1.7B on vLLM 0.32.0 needs `--gpu-memory-utilization 0.92 --max-num-seqs 32` for 8k context (10,688 KV tokens).
- Prefix caching cut p95 TTFT by 90-96.5% on agent-shaped traffic.
- AgentDesk evals: 76/100 baseline; phase 3 ablation 66 -> 77 -> 81 -> 79 (`1959d2a`); simulator now the main error source.
- LocalArena (`~/projects/localArena`) all 8 weeks complete; see [[Local-Arena|Local Arena]].

## Recent Changes

- Updated [[AgentDesk]] with phase 3.

## Active Threads

- AgentDesk next: reliable simulated customer (PLAN 3.7), then phase 4 (FlyCo/SaasCo tenants, LoRA).
- Open LocalArena PM-002 actions: make-based runbooks, region-aware reconnect, e2e rewards check in CI.
