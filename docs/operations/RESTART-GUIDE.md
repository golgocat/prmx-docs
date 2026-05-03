# PRMX Infrastructure Restart Guide

> **Scope**: restart procedures for PRMX local development and PRMX-side cloud services.

The canonical zero-start runbook for fresh chain bring-up lives in the source repo as an operator runbook and is not part of this published reading set. The default shared environment is the DO-hosted PRMX test chain + Base Sepolia, not `localhost`.

## TL;DR — what to run when

| Situation | Action |
|---|---|
| Local dev — temporary chain (default) | `./scripts/restart-dev-environment.sh` |
| Local dev — chain data survives restarts | `--persistent` |
| After a `git pull` or pallet change | `--build` |
| Runtime upgrade only (no data wipe) | `node scripts/council-runtime-upgrade.mjs <wasm>` |
| Public testnet endpoints (read-only reader) | See [Public Testnet](#public-testnet-interacting-from-outside) |
| Stuck Hyperlane message | See [Troubleshooting](#troubleshooting) |

> **Settlement note**: oracle finality ≠ payout finality. A policy can sit in `pending settlement` while PRMX prepares local liquidity. Restart and incident procedures must check both oracle health and capital/liquidity readiness.

## Architecture baseline

| Component | Stack |
|---|---|
| Database | Supabase (PostgreSQL) |
| Weather provider | Open-Meteo (free, no API key) |
| Pallets | `prmxOracleV4`, `prmxPolicyV4`, `prmxMarketV4`, `prmxOrderbookLpV4` |
| Pricing | Open-Meteo observed catalog (Cloud Run) |
| Products | 14 (see [Product Catalog](/docs/product/V4-PRODUCT-CATALOG)) |
| Locations | 40 Rainguard cities across 9 regions |

You need `INGEST_HMAC_SECRET` for OCW ingest auth. No AccuWeather key required.

## Contents

1. [Quick Start](#quick-start)
2. [OCW Secrets](#ocw-secrets)
3. [Post-Restart Steps](#post-restart-steps)
4. [Restart Modes](#restart-modes)
5. [Forkless Upgrade Safety](#forkless-upgrade-safety)
6. [What Happens When You Restart](#what-happens-when-you-restart)
7. [Secret Security](#secret-security)
8. [Services Overview](#services-overview)
9. [Public Testnet (interacting from outside)](#public-testnet-interacting-from-outside)
10. [Troubleshooting](#troubleshooting)

---

## Quick Start

Use the commands below for local-only development or PRMX-side service recovery. For collaborative testing against the shared PRMX test chain, use the shared integration runbook as the source of truth.

If the shared integration EVM stack changes, update or regenerate:

- `evm/abi/deployments/shared-integration.latest.json`
- `evm/abi/deployments/shared-integration.inventory.md`
- `evm/abi/deployments/shared-integration.frontend.env`
- `evm/abi/deployments/shared-integration.offchain.env`

Those artifacts are the source of truth for shared integration addresses. This guide is not.

```bash
# Temporary mode (recommended for development)
INGEST_HMAC_SECRET="your_32_char_secret" \
./scripts/restart-dev-environment.sh

# Persistent mode (data survives restarts)
INGEST_HMAC_SECRET="your_32_char_secret" \
./scripts/restart-dev-environment.sh --persistent

# Build before start (after pull or pallet changes)
INGEST_HMAC_SECRET="your_32_char_secret" \
./scripts/restart-dev-environment.sh --build

# Full example with DAO auto-underwrite
INGEST_HMAC_SECRET="your_secret" \
DAO_UNDERWRITE_ENABLED=true \
DAO_MNEMONIC="//Alice" \
./scripts/restart-dev-environment.sh
```

NOTE: `INGEST_HMAC_SECRET` authenticates the V4 OCW with the Oracle Service via HMAC. Open-Meteo requires no API key.

---

## OCW Secrets

The V4 OCW requires secrets stored in offchain worker persistent storage:

### Storage Locations

| Secret | Storage Key | Purpose |
|--------|-------------|---------|
| HMAC Secret | `ocw:v4:ingest_hmac_secret` | Authenticating with the Ingest API |
| Ingest URL | `ocw:v4:ingest_api_url` | URL of the off-chain oracle service |

NOTE: OCW storage keys use the `ocw:v4:` prefix.

### After Restart

After a `--tmp` restart, secrets are automatically injected if environment variables are set.
If OCW auth is failing, manually inject:

```bash
node scripts/set-oracle-secrets.mjs \
    --hmac-secret "$INGEST_HMAC_SECRET"
```

The `--ingest-url` flag defaults to `http://localhost:3001` and is usually not needed.

---

## Post-Restart Steps

After a `--tmp` restart, additional steps are required:

### 1. Populate Location Registry (40 Cities)

```bash
cd scripts && node populate-location-registry.mjs
```

Expected: "Successfully added: 40" locations. Without this, policy creation fails.

### 2. Add Oracle Member

```bash
node -e "
const { ApiPromise, WsProvider, Keyring } = require('@polkadot/api');
(async () => {
  const api = await ApiPromise.create({ provider: new WsProvider('ws://127.0.0.1:9944') });
  const dao = new Keyring({ type: 'sr25519' }).addFromUri('//DAO');
  const alice = new Keyring({ type: 'sr25519' }).addFromUri('//Alice');
  await api.tx.prmxOracleV4.addOracleMember(alice.address)
    .signAndSend(dao, ({ status, events }) => {
      if (status.isInBlock) {
        const added = events.some(e => e.event.method === 'OracleMemberAdded');
        console.log(added ? 'Oracle member added' : 'Already exists');
        api.disconnect();
      }
    });
})();
"
```

### 3. Insert Oracle Key into Keystore

```bash
node -e "
const { ApiPromise, WsProvider, Keyring } = require('@polkadot/api');
const { cryptoWaitReady } = require('@polkadot/util-crypto');
(async () => {
  await cryptoWaitReady();
  const api = await ApiPromise.create({ provider: new WsProvider('ws://127.0.0.1:9944') });
  const alice = new Keyring({ type: 'sr25519' }).addFromUri('//Alice');
  await api.rpc.author.insertKey('prmx', '//Alice', '0x' + Buffer.from(alice.publicKey).toString('hex'));
  console.log('Oracle key inserted');
  await api.disconnect();
})();
"
```

Without steps 2 + 3, V4 snapshots will NOT be submitted.

---

## Restart Modes

### Temporary Mode (default)

```bash
./scripts/restart-dev-environment.sh --tmp
```

- Creates a fresh chain each time (genesis regenerated)
- All on-chain data is lost
- Supabase service persists (remote database), but runtime tables can be cleared by oracle restart detection
- Best for: daily development, testing features

### Persistent Mode

```bash
./scripts/restart-dev-environment.sh --persistent
```

- Chain data stored in `$PRMX_DATA_DIR` (default: `/tmp/prmx-data`)
- On-chain data survives restarts
- Best for: multi-session testing, integration tests
- Required mode for forkless runtime upgrades

### Build Mode

```bash
./scripts/restart-dev-environment.sh --build
```

- Builds `prmx-node` (release) before starting
- Use after `git pull` or pallet changes to ensure V4 runtime binary
- Can combine with `--tmp` or `--persistent`

---

## Forkless Upgrade Safety

For forkless upgrades, use persistent chain data and preserve chain continuity.

### Safe Procedure (`--persistent`)

```bash
# 1) Build latest binary
cargo build --release -p prmx-node

# 2) Restart node with persistent base-path
#    (project-managed hosts use the equivalent service-restart command)
./scripts/restart-dev-environment.sh --persistent

# 3) Confirm node is healthy and producing blocks
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"system_health","params":[],"id":1}' \
  http://127.0.0.1:9944

# 4) Then restart oracle (or its managed service equivalent)
```

### Prohibited During Forkless Upgrade

- Do not run `--tmp`.
- Do not run `purge-chain`.
- Do not delete the persistent chain base-path.
- Do not manually `TRUNCATE observations/snapshots/timeline_events/lp_trades` unless intentionally resetting off-chain history.

### Why

Oracle startup checks `chain_meta`. If restart conditions are met, `clearAllData()` runs and clears runtime tables.

- Condition 1: `genesis_hash` changed
- Condition 2: block number appears reset (`current < last - 10`)

In a normal forkless upgrade, these conditions should never be true.

---

## What Happens When You Restart

### Temporary Mode Sequence

1. **Kill processes**: Stops any running node, oracle service, frontend
2. **Start blockchain node**: `prmx-node --dev --tmp` (fresh genesis)
3. **Wait for node**: Polls until block production begins
4. **Inject OCW secrets**: HMAC secret + ingest URL via RPC
5. **Start oracle service**: Connects to node (WS) + Supabase
6. **Start frontend**: Next.js dev server on port 3000

### What Gets Reset

| Component | Temporary Mode | Persistent Mode |
|-----------|---------------|-----------------|
| Chain data | **Wiped** | Preserved |
| On-chain state | **Wiped** | Preserved |
| OCW offchain storage | **Wiped** (re-injected) | Preserved |
| Oracle membership | **Wiped** (re-add) | Preserved |
| Location registry | **Wiped** (re-populate) | Preserved |
| Supabase data | Remote DB persists, but runtime tables may be cleared by oracle restart detection | Remote DB persists, but runtime tables may be cleared by oracle restart detection |

NOTE: Supabase service persistence is different from table contents. `observations`, `snapshots`, `timeline_events`, and `lp_trades` can still be cleared by `clearAllData()` when chain restart conditions are detected.

---

## Secret Security

### How Secrets Flow

```
Environment Variable          ->  Restart Script  ->  OCW Offchain Storage
INGEST_HMAC_SECRET                 set-oracle-secrets.mjs    ocw:v4:ingest_hmac_secret
```

### Security Properties

- **No hardcoded secrets**: All secrets come from environment variables
- **Offchain storage**: Secrets stored in Substrate's offchain persistent DB, not on-chain
- **No API keys for weather**: Open-Meteo is free (unlike AccuWeather in V3)
- **HMAC authentication**: OCW authenticates to Oracle Service via HMAC-SHA256

### Secret Locations

| Environment | Secret Storage |
|-------------|---------------|
| Local dev | `.env` file in project root |
| Project-managed hosts | OS-level secret store (path is operator-internal) |
| Supabase | Vercel env vars / oracle service env |
| Pricing API key | Cloud secret manager |

---

## Services Overview

### Local Development

| Service | Port | Log File | Purpose |
|---------|------|----------|---------|
| Blockchain Node | ws://localhost:9944 | /tmp/prmx-node.log | Substrate node + V4 OCW |
| Oracle Service | http://localhost:3001 | /tmp/oracle-service.log | Ingest API + DAO |
| Frontend | http://localhost:3000 | /tmp/frontend.log | Next.js UI |

### External Services

| Service | URL | Purpose |
|---------|-----|---------|
| Supabase | `https://<project-ref>.supabase.co` | PostgreSQL database (project URL injected from secrets) |
| Pricing API | Internal Cloud Run endpoint | Premium / payout pricing (auth required) |
| Open-Meteo | https://api.open-meteo.com | Weather data (free, no key) |

### DAO Auto-Underwrite

The DAO service (part of Oracle Service) automatically accepts insurance requests:

1. Listens for `RequestCreated` events on-chain
2. Validates against location whitelist and event type whitelist
3. Queries Pricing API for recommended premium
4. If premium is within tolerance, submits `acceptRequest` on-chain
5. Records decisions in Supabase `dao_underwrite_records`

Configuration (in `oracle-service/.env`):

```env
DAO_UNDERWRITE_ENABLED=true
DAO_MNEMONIC=//Alice
DAO_LOCATION_WHITELIST=0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31,32,33,34,35,36,37,38,39
DAO_EVENT_TYPE_WHITELIST=PrecipSumGte,Precip1hGte,Precip12hMaxGte,TempMaxGte,TempMinLte,WindGustMaxGte,PrecipTypeOccurred,HeatIndexMaxGte,SnowDepthMaxGte,PressureDropMaxGte,SunshineDurationSumLte,RiverDischargeMaxGte,WaveHeightMaxGte,Pm25MaxGte
DAO_PREMIUM_TOLERANCE=0.01
DAO_MAX_SHARES_PER_REQUEST=1000
DAO_SHARE_PERCENT=0.5
PRICING_API_URL=<pricing-api-url>
PRICING_API_KEY=<pricing-api-key>
```

---

## Public Testnet (interacting from outside)

Readers of this guide are expected to **interact with** the shared PRMX testnet rather than deploy their own. The infra topology below is informational only — host details, SSH access, and systemd service control are restricted to project operators.

| Surface | Endpoint | Reader use |
|---|---|---|
| Frontend | `https://prmx-v4.vercel.app` | Connect a wallet, run a policy lifecycle |
| Substrate WS | Published in the frontend's runtime config | Read chain state via Polkadot.js / RPC |
| REST API | Published in the frontend's runtime config | Read off-chain timeline / reports |

The Substrate node, oracle service, and reverse proxy run on a project-controlled host fronted by a Vercel-managed app. Restart and credential rotation are operator tasks; they do not affect external readers beyond brief endpoint downtime.

If the public endpoints are unreachable, check the [PRMX status page or Linear](https://linear.app/southernbreeze/team/SBP) before assuming a code-side failure on your end.

---

## Troubleshooting

### Off-Chain History Unexpectedly Reset

If observations/timeline counters suddenly drop after restart, check oracle startup logs first:

```bash
rg -n "Chain restart detected|Clearing all tables" /tmp/oracle-service.log
```

(On project-managed hosts the equivalent is `journalctl -u prmx-oracle`.)

If these messages appear during a forkless upgrade, treat it as an operational incident:

1. Verify node is using the same persistent base-path.
2. Verify no `purge-chain` or manual table `TRUNCATE` was run.
3. Restart in safe order: node healthy first, then oracle.

### OCW Auth Verification Failed

```bash
# Re-inject HMAC secret (load from your local secret store, then run)
cd scripts
node set-oracle-secrets.mjs \
    --hmac-secret "$INGEST_HMAC_SECRET"

# Reset backoff counter
node -e "
const { ApiPromise, WsProvider } = require('@polkadot/api');
(async () => {
  const api = await ApiPromise.create({ provider: new WsProvider('ws://127.0.0.1:9944') });
  const key = '0x' + Buffer.from('ocw:v4:auth_verification').toString('hex');
  await api.rpc.offchain.localStorageSet('PERSISTENT', key,
    '0x0100000000000000000000000000000000000000000000');
  console.log('Backoff reset');
  await api.disconnect();
})();
"
```


### No Local Oracle Accounts

Run post-restart steps 2 and 3 (add oracle member + insert key).

### Location Not Found

```bash
node scripts/populate-location-registry.mjs
```

### DAO Not Running

```bash
curl http://localhost:3001/dao/status | jq

# Check env vars
grep DAO oracle-service/.env

# Ensure these exist:
# DAO_UNDERWRITE_ENABLED=true
# DAO_MNEMONIC=//Alice
```

### Pricing API Unavailable

If the Pricing API is unreachable, the DAO underwriter rejects all requests (no silent fallback). Verify the configured `PRICING_API_URL` and that the project-issued key is still valid; if reachability is broken from a project-managed host, escalate to ops.

### Port Already in Use

```bash
lsof -ti:9944 | xargs kill -9   # Node
lsof -ti:3001 | xargs kill -9   # Oracle
lsof -ti:3000 | xargs kill -9   # Frontend
```

---

## Environment Variables Reference

### Oracle Service

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `INGEST_HMAC_SECRET` | Yes | -- | HMAC auth for OCW ingest |
| `SUPABASE_URL` | Yes | -- | Supabase instance URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Yes | -- | Supabase admin key |
| `WS_URL` | Yes | ws://localhost:9944 | Chain WebSocket |
| `REPORTER_MNEMONIC` | Yes | -- | Oracle reporter account |
| `API_PORT` | No | 3001 | REST API port |
| `INGEST_DEV_MODE` | No | -- | Disable auth (dev only) |

### DAO Auto-Underwrite

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DAO_UNDERWRITE_ENABLED` | No | true | Enable auto-underwrite |
| `DAO_MNEMONIC` | No | //Alice | DAO wallet mnemonic |
| `DAO_LOCATION_WHITELIST` | No | 0 | Comma-separated location IDs (0-39 for all 40 cities) |
| `DAO_EVENT_TYPE_WHITELIST` | No | (all 14 products) | Comma-separated product types |
| `DAO_PREMIUM_TOLERANCE` | No | 0.01 | Premium tolerance (1%) |
| `DAO_MAX_SHARES_PER_REQUEST` | No | 1000 | Max shares per underwrite |
| `DAO_SHARE_PERCENT` | No | 0.5 | Fraction of shares DAO takes (rest for LPs) |
| `DAO_MIN_PREMIUM_PER_SHARE` | No | 1000000 | Minimum premium per share (1 USDC) |
| `DAO_SETTLEMENT_ASSET_ID` | No | 1 | On-chain settlement asset ID |
| `PRICING_API_URL` | No | -- | Pricing API URL (primary) |
| `PRICING_API_KEY` | No | -- | Pricing API key |

### Frontend

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `NEXT_PUBLIC_WS_ENDPOINT` | Yes | ws://localhost:9944 | Chain WebSocket. Shared integration should use the public WSS endpoint from the shared runbook |
| `NEXT_PUBLIC_ENABLE_DEV_ACCOUNTS` | No | -- | Enable dev accounts (//Alice, etc.) |
| `PRICING_API_URL` | No | -- | Pricing API URL |
| `PRICING_API_KEY` | No | -- | Pricing API key |
| `NEXT_PUBLIC_ORACLE_SERVICE_URL` | No | -- | Oracle service base URL |

