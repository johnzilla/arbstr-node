# arbstr node

Full-stack deployment of [arbstr](https://arbstr.com): the cheapest route to any model, metered in sats.

One `docker compose up` brings up the complete stack: routing engine, treasury, Lightning, and Cashu mint.

> [!CAUTION]
> **DANGER: Do Not Run This as a Production Lightning Node**
>
> This project is experimental and under active development. It is **not hardened, audited, or production-ready**.
>
> Running a Lightning node involves **real financial risk**. Misconfiguration, bugs, or incomplete understanding can result in **permanent loss of funds**.
>
> Do **not** use this software with real Bitcoin unless you fully understand the Lightning Network, have reviewed the code, and are prepared to lose all funds involved.
>
> For experimentation, use **testnet, signet, or regtest only**.
>
> **You are solely responsible for any loss of funds.**

## Services

| Service | Host port | Description |
|---------|-----------|-------------|
| **core** | 8080 (all interfaces) | [arbstr](https://github.com/johnzilla/arbstr) routing engine — OpenAI-compatible proxy with cost/latency/capability arbitrage |
| **vault** | 127.0.0.1:3010 | [arbstr-vault](https://github.com/johnzilla/arbstr-vault) treasury — agent sub-accounts, Cashu + Lightning payment rails, policy enforcement, audit log (container port 3000 remapped to host 3010) |
| **lnd** | 127.0.0.1:10009 (gRPC), 127.0.0.1:8180 (REST) | Lightning Network daemon |
| **mint** | 127.0.0.1:3338 | Nutshell Cashu mint (self-hosted) |

Only core, the client-facing API, listens on the network; its endpoints require agent or operator tokens. Vault's admin API, LND and the mint are reachable only from this machine. To administer a remote node, tunnel instead of opening ports, e.g. `ssh -L 3010:127.0.0.1:3010 your-server`.

## Versions

Every service runs a pinned image version. The exact versions live in one place, [`docker-compose.yml`](./docker-compose.yml), and each [arbstr-node release](https://github.com/johnzilla/arbstr-node/releases) lists the set it ships. `docker compose config --images` prints them for your checkout.

| Service | Image | Releases |
|---------|-------|----------|
| core | `ghcr.io/johnzilla/arbstr` | [arbstr](https://github.com/johnzilla/arbstr/releases) |
| vault | `ghcr.io/johnzilla/arbstr-vault` | [arbstr-vault](https://github.com/johnzilla/arbstr-vault/releases) |
| lnd | `lightninglabs/lnd` | [lnd](https://github.com/lightningnetwork/lnd/releases) |
| mint | `cashubtc/nutshell` | [nutshell](https://github.com/cashubtc/nutshell/releases) |

The arbstr images are multi-arch (amd64 + arm64) and carry signed build provenance: `gh attestation verify oci://ghcr.io/johnzilla/arbstr:<version> --owner johnzilla`. Dependabot proposes new versions as weekly PRs; nothing upgrades on its own.

## FAQ

[FAQ](FAQ.md) 

## Quick Start

```bash
git clone https://github.com/johnzilla/arbstr-node.git
cd arbstr-node

# Configure
cp .env.example .env
# Edit .env:
#   VAULT_ADMIN_TOKEN, VAULT_INTERNAL_TOKEN — generate each with: openssl rand -hex 32
#   ARBSTR_ANTHROPIC_API_KEY — key for the Anthropic provider in config.toml
#   ARBSTR_OPERATOR_TOKEN — optional; enables core's request log and stats endpoints
#   MINT_PRIVATE_KEY — required; compose refuses to start without it
# Optionally edit config.toml to add providers or enable [tokenstats]

# Start
docker compose up -d

# Verify
curl http://localhost:8080/health
curl http://localhost:8080/providers   # routable providers + market.skipped hints
curl http://localhost:3010/health
```

The stack runs with `WALLET_BACKEND=simulated` by default: balances are tracked in vault's ledger, but no real funds move.

### Create an agent

Every request is billed to an agent account in vault. Create one, fund it, and give it spending limits. New agents get a deny-all policy, so requests are refused until you set one.

```bash
ADMIN="Authorization: Bearer $VAULT_ADMIN_TOKEN"   # value from .env

# Register — returns agent_id and token (the token is shown once; keep it)
curl -s http://localhost:3010/agents -H "$ADMIN" -H 'Content-Type: application/json' \
  -d '{"name": "my-agent"}'

# Deposit 100,000 sats (simulated)
curl -s http://localhost:3010/agents/<agent_id>/deposit -H "$ADMIN" -H 'Content-Type: application/json' \
  -d '{"amount_msat": 100000000}'

# Allow up to 50 sats per request and 1,000 sats per day
curl -s -X PATCH http://localhost:3010/operator/agents/<agent_id>/policy -H "$ADMIN" -H 'Content-Type: application/json' \
  -d '{"max_transaction_msat": 50000, "daily_limit_msat": 1000000}'
```

### Send a request

Point any OpenAI-compatible client at `http://localhost:8080`, using the agent's token as the API key:

```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer <agent token>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet-5-5",
    "max_tokens": 200,
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

Set `max_tokens`. Without it, core reserves for 4,096 output tokens at frontier rates, which can exceed the agent's per-request limit. A request the policy refuses returns `403` with an `error.reason` such as `exceeds_max_transaction` or `exceeds_daily_limit`.

### Request logs and stats

Core's `/v1/requests` (request log) and `/v1/stats` (cost and latency stats) cover every agent's traffic, so they take a separate operator token, not an agent token. Set `ARBSTR_OPERATOR_TOKEN` in `.env` (e.g. `openssl rand -hex 32`) and recreate core:

```bash
docker compose up -d --force-recreate core
curl -s http://localhost:8080/v1/stats -H "Authorization: Bearer $ARBSTR_OPERATOR_TOKEN"
```

Until it is set, both return `401 operator_token_not_configured`. `/v1/models` and `/v1/cost` take the agent token, like chat completions.

## How It Works

```
Your App ──> arbstr core ──(reserve)──> arbstr vault (agent account)
                  │       <─(settle / release)─
                  ├──> mesh-llm node (local)
                  ├──> Anthropic, Routstr, OpenRouter (static config or tokenstats)
                  └──> any OpenAI-compatible endpoint
```

1. **Request arrives** at arbstr core (port 8080) with the agent's bearer token.
2. **Vault reserves** the estimated cost against the agent's balance, after checking its policy (per-request and daily limits).
3. **Core routes** to the cheapest qualified **routable** provider (static config and/or live tokenstats quotes).
4. **Response streams** back to the client.
5. **Vault settles** the actual cost and refunds the rest of the reservation. If the provider fails, core releases it instead.

If core never settles a reservation (for example, it crashed mid-request), vault releases it after 15 minutes. Paying providers out of collected funds is not built yet.

## Configuration

### Environment (`.env`)

| Variable | Required | Description |
|----------|----------|-------------|
| `VAULT_ADMIN_TOKEN` | Yes | Operator admin token for vault (min 32 chars) |
| `VAULT_INTERNAL_TOKEN` | Yes | Shared secret between core and vault (min 32 chars) |
| `ARBSTR_ANTHROPIC_API_KEY` | For Anthropic | Key for the `anthropic` provider in `config.toml` |
| `ARBSTR_OPERATOR_TOKEN` | No | Bearer token for core's `/v1/stats` and `/v1/requests`. While unset, they refuse every request |
| `MINT_PRIVATE_KEY` | Yes | Key that signs the mint's ecash (`openssl rand -hex 32`). No default; changing it invalidates ecash already issued |
| `WALLET_BACKEND` | No | Vault payment rail: `simulated` (default), `lightning`, `cashu`, or `auto` |
| `LND_CERT_BASE64` / `LND_MACAROON_BASE64` | If `lightning`/`auto` | LND credentials for vault |
| `CASHU_THRESHOLD_MSAT` | No | Below: Cashu (instant). Above: Lightning. Default 1M msats |
| `OPERATOR_WEBHOOK_URL` | No | Where vault POSTs operator events (approval requests and outcomes, withdrawal requests, balance alerts) |
| `OPERATOR_WEBHOOK_SECRET` | No | HMAC-SHA256 key (min 16 chars) for the `X-ArbstrVault-Signature` header; unsigned if unset |
| `BILLING_RESERVATION_TTL_MS` | No | Age at which vault releases a reservation core never settled. Default 900000 (15 min), min 60000 |
| `RUST_LOG` | No | Core log level. Default `arbstr=info` |
| `OPENROUTER_API_KEY` | No | For `[tokenstats.keys] openrouter` / OpenRouter routes |
| `ROUTSTR_API_KEY` | No | Cashu or API key for a static Routstr provider |

### Rates (sats per 1M tokens)

Core prices providers in **sats per 1 million tokens** (RIP-05 / tokenstats).  
Legacy per-1k configs: multiply by 1000 (e.g. `10` sats/1k → `10000` sats/1M).

### Providers (`config.toml`)

Static providers are always available. Each is an OpenAI-compatible endpoint and needs a URL, models (or `auto_discover`), tier, and rates. The shipped `config.toml` has one active provider, Anthropic:

```toml
[[providers]]
name = "anthropic"
url = "https://api.anthropic.com/v1"
api_key = "${ARBSTR_ANTHROPIC_API_KEY}"
models = ["claude-sonnet-5-5"]
tier = "standard"           # local | standard | frontier
input_rate = 10000          # sats per 1M input tokens
output_rate = 30000         # sats per 1M output tokens
base_fee = 1                # sats per request
```

A local mesh-llm node needs no key and costs nothing:

```toml
[[providers]]
name = "mesh-local"
url = "http://host.docker.internal:9337/v1"
auto_discover = true
tier = "local"
input_rate = 0
output_rate = 0
```

### Live market feed (`[tokenstats]`) — optional, recommended

Uncomment the `[tokenstats]` block in `config.toml` to poll [tokenstats](https://tokenstats.ai) for multi-source quotes (OpenRouter, Routstr, …). Live pricing refreshes on an interval; market endpoints are **only routable when a key is configured**.

```toml
[tokenstats]
url = "https://tokenstats.ai"
poll_interval_secs = 60
allowed_hosts = ["openrouter.ai"]   # market endpoints must be https on these hosts to get your keys

[tokenstats.keys]
openrouter = "${OPENROUTER_API_KEY}"
# "node-provider-id" = "${ROUTSTR_API_KEY}"
```

Inspect routing readiness:

```bash
curl -s http://localhost:8080/providers | jq '.market'
# skipped[] explains missing_api_key, low_reliability, stale_quotes, …
```

Full options (policies, rate limiting, auth, vault): [arbstr core README](https://github.com/johnzilla/arbstr#configuration).

## Updating

```bash
# config.toml edits: recreate (not restart) — the file is bind-mounted, and
# editors that replace the file leave a restarted container on the old copy
docker compose up -d --force-recreate core

# New core or vault release: change its image tag in docker-compose.yml, then
docker compose pull && docker compose up -d
```

## Without Vault (Free Proxy Mode)

To run arbstr core standalone without billing, remove the `[vault]` section from `config.toml` and start core without its dependencies. Requests then need no agent token.

```bash
docker compose up -d --no-deps core
```

## Working on this repo

See [`CLAUDE.md`](./CLAUDE.md) for architecture notes, the release workflow, and common commands — written for agents but useful for human contributors too.

## Related Projects

- [arbstr](https://github.com/johnzilla/arbstr) -- Routing engine: OpenAI-compatible proxy that picks the cheapest qualified provider
- [arbstr-vault](https://github.com/johnzilla/arbstr-vault) -- Treasury: agent accounts, spending policy, per-request billing, Lightning and Cashu rails
- [arbstr-node](https://github.com/johnzilla/arbstr-node) -- The full stack (core + vault + LND + Cashu mint) as one `docker compose up`, pinned to released versions
- [mesh-llm](https://docs.anarchai.org) -- Distributed P2P inference network (local provider)
- [Routstr](https://routstr.com) -- Decentralized LLM marketplace

## License

MIT
