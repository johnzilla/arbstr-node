# arbstr node

Full-stack deployment of the [arbstr](https://arbstr.com) inference marketplace.

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
| **core** | 8080 | [arbstr](https://github.com/johnzilla/arbstr) routing engine — OpenAI-compatible proxy with cost/latency/capability arbitrage |
| **vault** | 3010 | [arbstr-vault](https://github.com/johnzilla/arbstr-vault) treasury — agent sub-accounts, Cashu + Lightning payment rails, policy enforcement, audit log (container port 3000 remapped to host 3010) |
| **lnd** | 10009 (gRPC), 8180 (REST) | Lightning Network daemon |
| **mint** | 3338 | Nutshell Cashu mint (self-hosted) |

## FAQ

[FAQ](FAQ.md) 

## Quick Start

```bash
git clone --recurse-submodules https://github.com/johnzilla/arbstr-node.git
cd arbstr-node

# Configure
cp .env.example .env
# Edit .env — set VAULT_ADMIN_TOKEN and VAULT_INTERNAL_TOKEN

# Edit config.toml — add providers and optionally enable [tokenstats]
# Edit .env — set vault secrets; optionally OPENROUTER_API_KEY for live market routes

# Start
docker compose up -d

# Verify
curl http://localhost:8080/health
curl http://localhost:8080/providers   # routable providers + market.skipped hints
curl http://localhost:3010/health
```

Point any OpenAI-compatible client at `http://localhost:8080`:

```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

## How It Works

```
Your App ──> arbstr core ──> arbstr vault (debit buyer)
                  │
                  ├──> mesh-llm node (local)
                  ├──> Routstr / OpenRouter (via static config or tokenstats)
                  └──> any OpenAI-compatible endpoint
                  │
             arbstr vault (credit seller) ──> Lightning payout
```

1. **Request arrives** at arbstr core (port 8080)
2. **Vault debits** the buyer's agent account (reserve pattern)
3. **Core routes** to the cheapest qualified **routable** provider (static config and/or live tokenstats quotes)
4. **Response streams** back to the client
5. **Vault settles** actual cost, refunds any overage

## Configuration

### Environment (`.env`)

| Variable | Required | Description |
|----------|----------|-------------|
| `VAULT_ADMIN_TOKEN` | Yes | Operator admin token for vault (min 32 chars) |
| `VAULT_INTERNAL_TOKEN` | Yes | Shared secret between core and vault |
| `MINT_PRIVATE_KEY` | No | Cashu mint key (auto-generated if unset) |
| `CASHU_THRESHOLD_MSAT` | No | Below: Cashu (instant). Above: Lightning. Default 1M msats |
| `RUST_LOG` | No | Core log level. Default `arbstr=info` |
| `OPENROUTER_API_KEY` | No | For `[tokenstats.keys] openrouter` / OpenRouter routes |
| `ROUTSTR_API_KEY` | No | Cashu or API key for a static Routstr provider |

### Rates (sats per 1M tokens)

Core prices providers in **sats per 1 million tokens** (RIP-05 / tokenstats).  
Legacy per-1k configs: multiply by 1000 (e.g. `10` sats/1k → `10000` sats/1M).

### Providers (`config.toml`)

Static providers are always available. Each needs a URL, models (or `auto_discover`), tier, and rates:

```toml
[[providers]]
name = "mesh-local"
url = "http://host.docker.internal:9337/v1"
auto_discover = true
tier = "local"              # local | standard | frontier
input_rate = 0              # sats per 1M input tokens
output_rate = 0             # sats per 1M output tokens
```

### Live market feed (`[tokenstats]`) — optional, recommended

Uncomment the `[tokenstats]` block in `config.toml` to poll [tokenstats](https://tokenstats.ai) for multi-source quotes (OpenRouter, Routstr, …). Live pricing refreshes on an interval; market endpoints are **only routable when a key is configured**.

```toml
[tokenstats]
url = "https://tokenstats.ai"
poll_interval_secs = 60

[tokenstats.keys]
openrouter = "${OPENROUTER_API_KEY}"
# "node-provider-id" = "${CASHU_OR_API_KEY}"
```

Inspect routing readiness:

```bash
curl -s http://localhost:8080/providers | jq '.market'
# skipped[] explains missing_api_key, low_reliability, stale_quotes, …
```

Full options (policies, rate limiting, auth, vault): [arbstr core README](https://github.com/johnzilla/arbstr#configuration).

## Without Vault (Free Proxy Mode)

To run arbstr core standalone without billing, remove the `[vault]` section from `config.toml` and run just the core service:

```bash
docker compose up core
```

## Working on this repo

See [`CLAUDE.md`](./CLAUDE.md) for architecture notes, the submodule workflow, and common commands — written for agents but useful for human contributors too.

## Related

- [arbstr](https://github.com/johnzilla/arbstr) — Routing engine
- [arbstr-vault](https://github.com/johnzilla/arbstr-vault) — Treasury service
- [mesh-llm](https://docs.anarchai.org/) — Block's distributed inference network

## License

MIT
