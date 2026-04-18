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

## Quick Start

```bash
git clone --recurse-submodules https://github.com/johnzilla/arbstr-node.git
cd arbstr-node

# Configure
cp .env.example .env
# Edit .env — set VAULT_ADMIN_TOKEN and VAULT_INTERNAL_TOKEN

# Edit config.toml — add your providers (mesh-llm, Routstr, etc.)

# Start
docker compose up -d

# Verify
curl http://localhost:8080/health
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
                  ├──> Routstr (marketplace)
                  └──> any OpenAI-compatible endpoint
                  │
             arbstr vault (credit seller) ──> Lightning payout
```

1. **Request arrives** at arbstr core (port 8080)
2. **Vault debits** the buyer's agent account (reserve pattern)
3. **Core routes** to the cheapest qualified provider
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

### Providers (`config.toml`)

Add providers to `config.toml`. Each provider needs a URL, model list, tier, and pricing:

```toml
[[providers]]
name = "mesh-local"
url = "http://host.docker.internal:9337/v1"
models = ["Qwen3-32B"]
tier = "local"           # local | standard | frontier
input_rate = 5           # sats per 1k input tokens
output_rate = 15         # sats per 1k output tokens
```

See the [arbstr config reference](https://github.com/johnzilla/arbstr#configuration) for full options including policies, rate limiting, and auth.

## Without Vault (Free Proxy Mode)

To run arbstr core standalone without billing, remove the `[vault]` section from `config.toml` and run just the core service:

```bash
docker compose up core
```

## Related

- [arbstr](https://github.com/johnzilla/arbstr) — Routing engine
- [arbstr-vault](https://github.com/johnzilla/arbstr-vault) — Treasury service
- [mesh-llm](https://docs.anarchai.org/) — Block's distributed inference network

## License

MIT
