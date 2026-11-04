---
type: session
title: "AgentDesk"
created: 2026-10-31
updated: 2026-10-31
status: developing
tags:
  - session
  - agentdesk
  - vllm
  - inference
  - agents
evidence: synthetic
---

# AgentDesk

> [!note] Provenance
> Build log for AgentDesk (repo `~/projects/agentdesk`). Numbers come from experiments on one laptop (Dell G15 5530, RTX 3050 6 GB Laptop GPU, 15 GB RAM). They are relative measurements on this hardware, not capacity claims. Session record, not independently verified evidence.

## What it is

A multi-tenant AI customer-support agent platform running small open models (Qwen3-1.7B) on a self-hosted vLLM stack, built to learn agentic AI, harness engineering, distributed LLM serving (vLLM, llm-d, vLLM Semantic Router) and running agents at scale. Three fictional tenants (ShopCo, FlyCo, SaasCo) with their own tools, policies and LoRA adapters. Success is graded on final database state.

Repo docs: `CONTEXT.md` (why/what), `docs/PLAN.md` (phases, patterns), `docs/COMMANDS.md` (every command with purpose).

## Decisions

- One system covering all topics instead of 12 separate demo projects.
- Customer support instead of coding agents: small models handle short tool-calling tasks well and long reasoning poorly.
- Three routing layers, each first hand-written then compared with the real project: model choice (vllm-sr), replica choice (llm-d), execution (vLLM).
- Native Docker Engine for GPU work; Docker Desktop on Linux can't pass the GPU through.

## Environment fixes (2026-10-23 to 2026-10-31)

- GPU unusable because the NVIDIA module existed only for kernel 7.0.0-34 while 7.0.0-38 was running. Fix: install `linux-modules-nvidia-595-open-7.0.0-38-generic`; use `apt full-upgrade` in future.
- Installed NVIDIA Container Toolkit 1.20.1 and Docker Engine 29.9.2 alongside Docker Desktop. `docker context default` = GPU-capable native engine.

## Phase 1: serving baseline (done 2026-10-31)

- vLLM 0.32.0, Qwen3-1.7B bf16: weights 3.22 GiB; `--gpu-memory-utilization 0.85` left 0.39 GiB KV cache and vLLM refused to start at 8k context. `0.92` + `--max-num-seqs 32` gives 10,688 KV tokens.
- Tool calling went from 0/5 to 5/5 after rewriting a vague tool description (also reduced to one tool, so cause not isolated).
- Go stdlib load generator (`loadgen/`) replays multi-turn agent sessions with a shared ~1.7k-token prompt and records TTFT/ITL.
- Experiment 01, prefix caching off vs on: p95 TTFT 737 → 75 ms (c=1), 1,951 → 197 ms (c=4), 15,464 → 535 ms (c=16); throughput at c=16 56 → 217 tok/s; ~97% of prompt tokens hit the cache. The c=16 cache-off collapse is probably KV-memory queueing (unverified).
- Commit `6e7132a`.

## Phase 2: ShopCo tenant and agent loop (done 2026-10-31)

- Postgres 18 (ShopCo schema, 15 orders each built to hit one policy rule or boundary), Python harness with `uv`: tool registry (Pydantic-validated arguments, errors returned to the model as JSON, identity from session not arguments), five tools with policy enforced in code and `SELECT ... FOR UPDATE`, hand-written agent loop with step/token/time limits and stop reasons, simulated customer, 20 tasks graded on final DB state (tau-bench style).
- 41 tests, including every refusal rule and loop limits with a fake LLM.
- Baseline (5 trials x 20 tasks): **76/100 pass, pass^5 14/20**; refusals 88%, actions 55%; 2,667 tokens and 2.7 s p50 per conversation.
- Main failure: the model announces an action ("let me check", "we offer a credit") without calling the tool (18 of 24 failures). Also misread "over 200" as "200 or more", hallucinated a 1,500 USD refund amount, and once set `item_unused: true` without the customer saying so, which let a wrong refund through.
- Code-level checks blocked 20 of 21 wrong write attempts.
- Commit `75052f1`.

## Phase 3: harness hardening (done 2026-10-31)

- Always on: refunds over 200 USD create a pending request (state machine pending -> approved/rejected, decided with a compare-and-set `UPDATE ... WHERE status = 'pending'`) plus an approver CLI; audit log of every tool call; idempotency keys so a repeated write in the same conversation replays the first result.
- Measured one at a time (100 conversations each): phase-2 prompt 66 -> prompt v2 77 -> announce-without-acting guard 81 -> tool-side "unused" check 79.
- Prompt v2 was the biggest win (actions 16 -> 27 of 45); sending large refunds through the tool fixed T06 (0/5 -> 5/5).
- The guard re-samples a reply that promises an action without a tool call, using `tool_choice="required"` (vLLM constrained decoding). It tripped 11 times per 100 conversations for +4 passes.
- Asking the model to quote the customer failed: the 1.7B model paraphrased. The tool now reads the customer's messages itself, with a hedge list after "I don't know if the item is unused" slipped through a substring match.
- About 20 of the 21 remaining failures come from the simulated customer (quits early, or contradicts its goal). Next: a reliable customer simulator.
- Commit `1959d2a`.

## Lessons

- KV-cache memory, not compute, limits capacity on a 6 GB GPU; prefix sharing is what lets more conversations run.
- Prefix caching only works if early prompt tokens are byte-identical: shared content first, per-request content last.
- Tool descriptions are harness code and need evals.
- Bash: `grep -m1` in a pipe plus `set -o pipefail` aborts scripts via SIGPIPE.
- psycopg 3: `conn.transaction()` with no transaction open is top-level and COMMITS. Wrap work in an outer `transaction(force_rollback=True)` so inner calls become savepoints.
- Enforce policy in tool code, not only the prompt; and never let a money decision depend on a fact the model asserts (`item_unused`).
- Small models fail more at completing actions than at refusing them.
- Don't ask a small model to supply a fact the system can read itself; it paraphrases or invents.
- Once the harness is decent, the eval's simulated user becomes the bottleneck; measure that before tuning more.
- `docker context` can change under you (Docker Desktop); pin `DOCKER_CONTEXT=default` in scripts.
