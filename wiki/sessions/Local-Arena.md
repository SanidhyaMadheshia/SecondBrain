---
type: session
title: "Local Arena"
created: 2026-10-18
updated: 2026-10-19
status: developing
tags:
  - session
  - localarena
  - system-design
  - kafka
  - postgres
evidence: synthetic
---

# Local Arena

> [!note] Provenance
> Summary of one working session building LocalArena (repo `~/projects/localArena`, GitHub `SanidhyaMadheshia/localArena`). Numbers come from experiments run on one laptop (i5-13450HX, Docker Desktop 8 GiB); they are relative, not capacity claims. This is a session record, not independently verified evidence.

## Outcome

- **Week 1** (commit `2843f06`): authoritative fixed-tick Go game server (gorilla/websocket), deterministic engine, Go bot swarm, Canvas client, Experiment 001, ADRs 0001-0003.
- **Week 2** (`a502f04`, `f957e1e`): PostgreSQL 18.6 + replica + PgBouncer (Compose) and CloudNativePG on kind; goose migrations; concurrency-safe purchases, lobbies, idempotency keys; shard router; Experiment 002, ADRs 0004-0006.
- **Week 3** (`604ab0e`, `66debc0`): Envoy edge over 3 API replicas, Valkey cache/rate limits/circuit breaker, timeout budget, Garage + 2 NGINX caching edges, k6 chaos, kind node-loss drill; Experiment 003, ADRs 0007-0008.
- **Week 4** (commit `38bbed5`): 3-broker KRaft Kafka, protobuf contracts in Apicurio, non-blocking Go producer with disk spool, Debezium outbox, Kafka Streams leaderboard and sessions (Java), Go reward saga with retry topic and DLQ; Experiment 004, ADRs 0009-0012.
- **Week 5** (commit `6c30103`): Keycloak OIDC + go-oidc JWKS verification (rotation-safe), sub-based accounts, DB bans, OPA fail-closed authz, /v2 API (v1 deprecated), HMAC game tickets, SOPS+age secrets, Envoy TLS/WAF/rate limit, signed replay-protected webhook, SSRF-safe avatar import, flagd rollout; scans clean; threat model; Experiment 005, ADRs 0013-0015.
- **Week 6** (commit `f13f84a`): OTel tracing (otelhttp, otelpgx) to Tempo, Loki + Alloy logs with trace ids, Prometheus RED/SLO rules with multi-window burn-rate alerts (promtool-tested), Alertmanager + lab pager, Grafana dashboard, runbooks, backup/restore and failover RPO drills, hidden-incident drill and postmortem PM-001; Experiment 006, ADRs 0016-0017.
- **Week 7** (commits `5a4af17`, `8202560`): gameserver graceful drain, Helm chart (CNPG, migrate pre-upgrade hook, HPA/PDB/canary), metrics-server + HPA, OpenTofu platform (cnpg, metrics-server, Argo CD; imports, no drift), GitHub Actions CI (actionlint clean), Argo CD GitOps with self-heal; rolling/canary/blue-green/failed-migration/rollback/drain drills all zero failed requests; Experiment 007, ADRs 0018-0020.
- **Week 8** (commit `081f214`): two simulated regions (Toxiproxy WAN 20/40 ms player, 150 ms inter-region; CoreDNS authoritative + caching resolver; region B gameserver/API with single-writer economy), client-side region selection, cost model per player/match, final game day with 3 hidden overlapping faults (all diagnosed), PM-002, Experiment 008, ADRs 0021-0022, `FINAL_REPORT.md`.
- **Docs** (commit `17087bb`): `docs/` study guide: architecture HLD, 22 ADR deep dives (HLD/LLD, 87 Mermaid diagrams validated with the mermaid 11 parser, concepts with "where else", code pointers, 15 interview Q&A each), 1,400-line glossary, interview cheatsheet. Written by 10 parallel agents plus a reviewer.
- **3D client + guide** (commit `07d6c1b`): `web/` rebuilt with Vite 8 + three.js 0.186 (shadows, gems, particles, labels, camera follow, cosmetics on the avatar), snapshot interpolation + client prediction/reconciliation, Keycloak login (PKCE) and account play via game tickets, shop with idempotent purchases, `/v2/leaderboard` (Streams top-N + names, degrades to 503). `make play` / `web-dev` / `web-smoke` (Playwright); `GETTING_STARTED.md` covers run, play, test, develop, troubleshoot and a launch checklist.

## Lessons worth keeping

### Runtime and networking (Week 1)
- Bounded application queues do not bound staleness: kernel socket buffers absorbed hundreds of snapshots, so slow clients were seconds behind with an empty mailbox. Fix: clients report the last processed tick and the server throttles lagging clients (never cut them off completely, which livelocks).
- Overload showed up as tick *start* lag (~99 ms) while simulation work stayed ~0.07 ms; the bottleneck was socket fan-out, not the simulation.

### Databases (Week 2)
- Naive read-then-write sold 200 items from 20 in stock; a CHECK constraint cannot catch lost updates.
- `SELECT ... FOR UPDATE` deadlocked every buyer because the earlier foreign-key insert already held `FOR KEY SHARE`; `FOR NO KEY UPDATE` fixed it. A single guarded `UPDATE ... WHERE stock > 0` was fastest.
- An index alone did not fix history latency; taking LIMIT before the join did (5.7 ms to 0.8 ms).
- CNPG failover: PgBouncer's open connections kept Postgres in smart shutdown (~3 min), then PgBouncer cached the login error for 15 s. `smartShutdownTimeout: 10` plus `server_login_retry: 1` brought the outage to ~1 s.

### Traffic and resilience (Week 3)
- Timeout order must be innermost-shortest: DB 2 s < Envoy per-try 2.5 s < API 3 s < route 4 s. The reverse made Envoy retry 7,482 times against one slow database.
- Do not let a proxy retry 503s caused by a shared dependency; the API marks them so Envoy skips retries.
- A hung replica is worse than a dead one; least-request routing avoided it (p99 12 ms vs 2.5 s round-robin).
- Request collapsing (singleflight) cut hot-key cache-miss DB loads from 650 to 62; stampedes need fill cost x request rate to show up.
- Node-loss drill found single points of failure: one CNPG operator pod, one pooler pod, both API pods on one node, and a `/readyz` that failed on DB loss (removing every endpoint). After fixes: ~60 s write outage, dominated by node-failure detection.
- MinIO was archived upstream in April 2026; Garage replaced it.

### Streaming (Week 4)
- franz-go measures delivery timeout from `Record.Timestamp`; replayed spool records stamped with old event time failed instantly forever. Fix: record timestamp = produce time, event time in the Envelope.
- RocksDB (Kafka Streams) needs glibc; Alpine JRE fails with `libstdc++.so.6` missing.
- Default 45 s consumer session timeout made Streams failover take 46 s; 10 s gave 11 s.
- A replay tool producing into the topic it reads must stop at start-time end offsets, or it chases its own output (it pushed ~1.45 M duplicates; credits and leaderboard stayed unchanged, confirming idempotency).
- Results: Kafka down 60 s during play: 0 lost, 0 duplicate durable events, tick p99 0.46 ms. Debezium stopped then killed mid-backlog: 200/200 purchases published, 0 duplicates. Breaking protobuf change (int64 to string) rejected by Apicurio.

### Security (Week 5)
- Swapping the last base64url character of a JWT is not a reliable forgery test: its low bits can be padding, so the signature bytes are unchanged. Edit the claims instead.
- Envoy route `safe_regex` matches only the path; match the `:path` header to include the query string.
- Keycloak answers a wrong password with 400 `invalid_grant` (OAuth2), not 401.
- Image scans must include base layers: the Streams image's Go stdlib CVEs came from Ubuntu's `pebble` binary.
- Check SSRF at dial time (resolved IP), not on the hostname, or DNS rebinding bypasses it.
- Bans belong in the database, checked per request, so they apply before tokens expire.
- The age key `~/.config/sops/age/keys.txt` must be backed up; the public recipient is in `.sops.yaml`.

### Observability and recovery (Week 6)
- A ratio SLI `1 - good/total` reads 100% bad when total is 0; it paged during the incident with no games running. Guard with `total > 0`.
- `sum(rate(x{code="5xx"}))` is empty (not 0) with no errors; use `or vector(0)` in recording rules.
- Async replica failover: RTO ~0.7 s, RPO = replication lag at failure (0 healthy; 139 acknowledged writes lost after a 5 s partition).
- Hidden incident (paused PgBouncer) was diagnosed in ~1 min because the slow trace had no SQL span; the missing signal was PgBouncer metrics.
- Version drift: Tempo 3 moved retention to `backend_worker`; the OTel collector needs an explicit Prometheus reader for its own metrics.

### Kubernetes delivery (Week 7)
- Helm 4 uses server-side apply: a manual `helm upgrade` on an Argo CD-managed app fails with field-ownership conflicts. One deployer per app; pause auto-sync for drills.
- `--rollback-on-failure` on a first install uninstalls everything without `helm.sh/resource-policy: keep` (the CNPG Pooler was deleted, the Cluster kept).
- OPA loading a ConfigMap directory reads the policy 3 times through `..data` symlinks ("multiple default rules"); load the file path.
- A failing migration in a pre-upgrade hook fails the release before new pods start; the old revision kept serving (0 failed requests, schema unchanged).
- Game servers need allocation (free slots, empty-server scale-in), not CPU HPA; drain on SIGTERM with grace > round length.

### Multi-region and operations (Week 8)
- Single-writer economy: a region B write took 805 ms vs 35 ms local, because every SQL round trip in the transaction crosses the 150 ms WAN. Batch writes into one round trip or give each player a home region.
- During a partition, B reads stayed 200 and B writes failed fast with 503 in 53 ms; gameplay was unaffected. Dead pooled connections surface as EOF/ECONNRESET, not SQLSTATE; classify them as unavailable, or they become 500s.
- Sticky clients made 1,549 failed reconnects when region A went down; re-running region selection gave 0 failures. DNS failover took ~34 s with a 30 s TTL (resolver cache), on top of 4-14 s for the authoritative reload.
- Egress dominates cost: ~15 KB/s per CCU (~39 GB/month). Below a few hundred CCU, the fixed HA platform costs more than the game.
- Game day: SLO burn-rate and PM-001's PgBouncer alert caught the DB fault; nothing alerted on stopped Streams processors. Added kafka-exporter plus `ConsumerLagGrowing`.
- Latent regression found on game day: once game tickets were enabled (week 5), `?player=` was ignored, so account bots silently became guests and **no rewards were granted for weeks**. Fix: an explicit lab flag `TRUST_PLAYER_PARAM` (default off, tested) and a business-outcome alert `NoRewardsGranted`. Lesson: alert on outcomes, not just component health.
- `docker kill` is not restarted under `restart: unless-stopped`. An ad-hoc `compose up` during mitigation dropped an env override (round length). Run mitigations from runbooks/make.
- CoreDNS images are distroless; on a Docker Desktop bind mount, rewrite zone files in place (`cat > file`) rather than with `sed -i`, which changes the inode.
- Mermaid: `;` inside sequence-diagram messages ends the statement. Validate docs with `mermaid.parse` under jsdom.

### Frontend (post Week 8)
- A CSS `display:` rule on an element overrides the `hidden` attribute; the hidden "no WebGL" overlay blocked every click. Add `[hidden] { display: none !important; }`.
- Headless Chrome renders WebGL with `--use-angle=swiftshader --enable-unsafe-swiftshader`, which is good enough for Playwright smoke tests and screenshots.
- New contributors don't have the age key: `secrets-env.sh` now generates random dev secrets instead of failing.

## Environment notes
- A host `redis-server` occupies 6379; Compose Valkey is on 6380.
- The `golang:1.27.2` Debian image was corrupt locally; race tests use `golang:1.27.2-alpine` + gcc.
- gosec 2.29.0 needed a rebuild against `golang.org/x/tools` v0.52.0 to analyse Go 1.27 code.
- kind cluster now has 2 workers; nodes were stopped during Week 4 to free memory.

## Open items
- All 8 weeks and docs are complete. Open PM-002 actions: runbook commands via make, region-aware reconnect in clients, e2e "reward granted" check in CI.
- Admin UIs are unauthenticated (loopback only).
