# Oracle Service Worker / Automation Surface

> **Scope**: Process layout, worker domains, env surface, and operational endpoints for the oracle service.

## Process modes

| Mode | When | What runs in-process |
|---|---|---|
| `MODE=monolith` (default) | Local dev, small deployments | API + capital watchers + reporters + monitors + operator endpoints |
| `MODE=workers` | Production-style fanout | Selected capital domains run as independent processes with Supabase-backed leader election |

Capital operations are Hyperlane-native: `VAULT_REPORTER_TRANSPORT=hyperlane` is the live posture; `both` is a rollback soak mode after a runtime or `YieldReportRecipient` redeploy.

Read-only Codex automations supplement the service — they should report deltas and first-suspected blockers, **never mutate** live capital state.

Architecture context: [V4 Oracle Architecture](/docs/architecture/V4-ORACLE-ARCHITECTURE).

## Worker processes

| Worker | `WORKER_NAME` | Domain | Health | Metrics |
|---|---|---|---|---|
| Deposit | `deposit` | Reconciles pending wallet links and Base → PRMX Hyperlane deposits into `pallet-assets(1)` | 8081 | 9091 |
| Exit | `exit` | Releases canonical `warpAccount.request_exit` exits on Base and finalizes them on PRMX | 8082 | 9092 |
| Settlement | `settlement` | Finalizes PRMX settlements after Base reserve-return evidence | 8083 | 9093 |
| Yield Reporter | `yield-reporter` | Vault asset reporting via Base `YieldReporter` → Hyperlane → PRMX `0x0802` | 8085 | 9095 |

## Runtime automation

These surfaces are usually owned by the monolith API process, even when the worker split is available.

| Surface | Purpose | Operator / readback |
|---|---|---|
| **Vault reporter** | Periodic + on-demand `PolicyVault.totalAssets()` reporting | `GET /capital/vault/status`, `POST /capital/operator/run-vault-report` |
| **Yield accrual driver** | ICA yield-command driver for testnet strategy accrual | `GET /capital/yield-accrual/status`, `POST /capital/operator/run-yield-accrual` |
| **Morpho borrower driver** | Keeps the testnet Morpho market utilized so vaults can observe borrow interest | `GET /capital/morpho-borrower/status`, `POST /capital/operator/run-morpho-borrower` |
| **Morpho live NAV probe** | Display-only Base RPC probe for policy `PolicyVault.totalAssets()` between canonical reports | `GET /capital/morpho-live-nav-probe/status`, `GET /capital/policies/:policyId/live-nav` |
| **Morpho vault migration** | Moves eligible vaults to the Morpho strategy surface during a migration window | `GET /capital/morpho-vault-migration/status`, `POST /capital/operator/run-morpho-vault-migration` |
| **Rebalancer (monitor / decision / executor)** | Classifies vaults, builds rebalance decisions, dispatches eligible drift through ICA | Hardened smoke `api-lifecycle-hyperlane`; executor logs must show ICA dispatch on production runs |
| **Warp invariant monitor** | Watches bridge-net supply vs Base collateral; persists history for alerts | `GET /capital/warp-invariant`, `warp_invariant_samples`, `oracle_warp_invariant_*` metrics |

> `POST /capital/operator/*` endpoints require `Authorization: Bearer $CAPITAL_OPERATOR_TOKEN`. Keep the token only in the host secret store — never in docs, manifests, logs, or screenshots.

## Running Workers

### npm scripts (local dev)

```bash
npm run worker:deposit
npm run worker:exit
npm run worker:settlement
npm run worker:yield-reporter
```

### Docker Compose

```bash
cd oracle-service && npm run build
docker build -f Dockerfile.worker -t prmx-oracle-worker .
docker compose -f docker-compose.workers.yml up -d
```

## Environment variables

### Process / leader election

| Variable | Default | Description |
|---|---|---|
| `MODE` | `monolith` | `monolith` or `workers` |
| `WORKER_NAME` | — | Required in worker mode: `deposit`, `exit`, `settlement`, `yield-reporter` |
| `WORKER_INSTANCE_ID` | `hostname-random` | Unique instance ID for leader election |
| `LEADER_HEARTBEAT_MS` | `30000` | Leadership heartbeat interval |
| `LEADER_STALE_MS` | `90000` | Stale-leader replacement threshold |
| `HEALTH_PORT` | `8081` | `/healthz` port (worker mode) |
| `METRICS_PORT` | `9090` | Prometheus `/metrics` port |
| `CAPITAL_OPERATOR_TOKEN` | unset | Required for operator-triggered capital endpoints |

### Vault reporter / capital watchers

| Variable | Default | Description |
|---|---|---|
| `VAULT_REPORTER_TRANSPORT` | `hyperlane` | Yield-report transport; `both` is rollback soak mode only |
| `VAULT_REPORTER_REBALANCE_ACK_INTERVAL_MS` | `30000` | Scan cadence for Base rebalance-ack evidence |
| `VAULT_REPORTER_REBALANCE_ACK_LOOKBACK_BLOCKS` | `5000` | Bounded Base log lookback for rebalance-ack evidence |
| `CAPITAL_WATCHER_SETTLEMENT_LOOKBACK_BLOCKS` | `5000` | Base reserve-return lookback for settlement recovery |
| `SETTLEMENT_DISPATCH_EVENT_LOOKBACK_BLOCKS` | `5000` | Override for settlement dispatch event recovery |

### Morpho NAV probe / rebalancer / invariant

| Variable | Default | Description |
|---|---|---|
| `MORPHO_LIVE_NAV_PROBE_ENABLED` | `false` | Display-only Base RPC NAV probes into `policy_live_nav` (never a settlement input) |
| `MORPHO_LIVE_NAV_PROBE_INTERVAL_MS` | `30000` | Cadence (≥ 5s) for the Morpho NAV probe |
| `REBALANCER_MONITOR_FUNDING_BASELINE_TOLERANCE_ATOMIC` | `1000000` | 1 USDC tolerance for funding-vs-drawdown classification |
| `WARP_INVARIANT_MONITOR_ENABLED` | `false` | Enables warp-invariant endpoint, samples, and metrics |

See `oracle-service/PRODUCTION.md` for the grouped production env surface.

## Leader election

Each worker type has at most one active leader in Supabase `worker_leadership`. Standby instances poll and take over after `LEADER_STALE_MS`.

## Endpoints

| Endpoint | Behavior |
|---|---|
| `GET /healthz` | `200` healthy, `503` if tick lag > 3× poll interval or leader changes > 5/hr |
| `GET /metrics` | Prometheus text format |

Core metrics:

- `oracle_worker_tick_duration_seconds{worker}`
- `oracle_worker_errors_total{worker}`
- `oracle_kind_dispatched_total{kind, result}`
- `oracle_relay_latency_seconds{direction}`
- `oracle_leader_changes_total{worker}`
- `oracle_pending_items{worker, type}`
- `oracle_warp_invariant_*`

## Operational notes

- **Zero-baseline funding/report guard** — when a vault is newly funded on Base but no rebalance-ack has landed, the vault reporter suppresses false yield/funding reports instead of minting synthetic accounting.
- **Vault-backed settlement** — the settlement worker uses the fresh `latestVaultAssets` snapshot as the Base reserve-return amount and **fails closed** if the snapshot is missing. PRMX ack is clamped to outstanding local liquidity. `PolicyVault → Reserve` is not a user payout.
- **Rebalancer classes** — `uncalibrated_or_unknown`, `already_funded`, `real_drawdown_candidate`. Only the last triggers an automatic rebalance-in batch.
- **Rebalancer production proof** — hardened `api-lifecycle-hyperlane` gate or targeted executor logs showing ICA dispatch. Direct Base signer fallback is local/dev only.
- **Surplus repair** — manual repair script, not a recurring worker. Dry-run first; execute only during an approved repair window; follow with `/capital/invariants` and the hardened lifecycle gate.
- **Codex `new-policy-monitor`** — automation surface at `scripts/prmx-monitor/new-policy-monitor.mjs`. Keeps cursor/snapshot state under `~/.codex/automations/prmx4-new-policy-monitor/memory.md`; checks Base `bridgedIn - returnedOut` overfunding, PRMX pool vs `maxPayout + reportedYield`, and vault-report freshness. Stale report evidence is a note (not a delivery anomaly) when periodic dispatch is intentionally disabled.
- **Shared signers** — in worker mode, multiple workers may share a signer; retry logic handles nonce contention. Avoid overlapping ad-hoc operator runs during a mutating smoke unless the gate explicitly expects it.
