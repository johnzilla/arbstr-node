## FAQ

<details>
<summary>What is arbstr-node?</summary>

arbstr-node is the full-stack, one-command deployment for the Arbstr inference marketplace.

It brings together:
- **arbstr core** (OpenAI-compatible routing + arbitrage engine)
- **arbstr-vault** (Lightning + Cashu treasury and billing)
- **LND** (Lightning Network daemon)
- **Nutshell Cashu mint** (self-hosted ecash mint)

One `docker compose up` gives you a complete local or self-hosted inference routing + payments stack.

</details>

<details>
<summary>What does it actually do?</summary>

It turns your local machine (or server) into a smart proxy that:
- Accepts OpenAI-compatible requests
- Automatically routes each request to the cheapest suitable **routable** provider (local models, Routstr/OpenRouter via config or live [tokenstats](https://tokenstats.ai), or custom endpoints)
- Prices in **sats per 1M tokens** (RIP-05); optional live market feed for multi-source quotes
- Handles payment reservation, settlement, and payouts in Bitcoin sats (via Lightning and Cashu)

Perfect for running your own mini inference marketplace or just getting cheap + private LLM access.

</details>

<details>
<summary>What is tokenstats and do I need it?</summary>

**Optional but recommended.** [tokenstats](https://tokenstats.ai) is a live inference price board. When you enable `[tokenstats]` in `config.toml`, arbstr core polls it and can route to market providers that have keys in `[tokenstats.keys]`.

Without it, only static `[[providers]]` in `config.toml` are used.  
`GET /providers` shows routable providers and why market sources were skipped (e.g. missing API key).

</details>

<details>
<summary>Do I need Bitcoin / Lightning to use it?</summary>

**No — payments are optional.**

You can run it in **free proxy mode** (no vault) for pure routing.  
Enable the vault + Lightning/Cashu when you want real sats-based arbitrage and provider payouts.

</details>

<details>
<summary>How do I get started?</summary>

```bash
git clone --recurse-submodules https://github.com/johnzilla/arbstr-node.git
cd arbstr-node
cp .env.example .env
# Edit .env and config.toml
docker compose up -d
```

See the [README](./README.md) for full configuration, tokenstats setup, and free-proxy mode.

</details>
