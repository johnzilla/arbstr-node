# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository shape

This is a **meta-repo** that composes a full-stack inference marketplace from four services. Only one of them has source here:

- `./` (root) — `docker-compose.yml`, `config.toml` (arbstr core config), `.env.example`. Orchestration only.
- `./vault/` — **git submodule** of [`arbstr-vault`](https://github.com/johnzilla/arbstr-vault) (Node.js/TypeScript/Fastify treasury service). This is where nearly all service code changes happen. **Commits to vault code go to the `arbstr-vault` repo, not this one.** This repo only records the pinned commit SHA via `.gitmodules`.
- **core** — pulled as `ghcr.io/johnzilla/arbstr:latest`. Rust routing engine. Source lives in [`johnzilla/arbstr`](https://github.com/johnzilla/arbstr), **not in this repo**. Don't look for Rust source here.
- **lnd** / **mint** — third-party images (`lightninglabs/lnd`, `cashubtc/nutshell`).

The untracked `vault/` in the parent directory (`/home/john/vault/projects/`) is unrelated — it's the user's personal projects vault, not this repo's `./vault/`.

### Submodule workflow

Fresh clone must init submodules or vault will be empty:
```bash
git clone --recurse-submodules <repo>
# or, after a plain clone:
git submodule update --init
```

Bump vault to a newer upstream commit:
```bash
git submodule update --remote vault
git add vault && git commit -m "build: bump vault submodule"
```

Working on vault code: `cd vault`, make edits, commit & push to `arbstr-vault` (origin is already set). Then back at the root, `git add vault` + commit to record the new pinned SHA here.

## Common commands

### Full stack (root)

```bash
cp .env.example .env                 # set VAULT_ADMIN_TOKEN and VAULT_INTERNAL_TOKEN (both ≥32 chars)
docker compose up -d                 # start core, vault, lnd, mint
docker compose logs -f core          # follow core logs
docker compose restart core          # pick up config.toml edits (mounted read-only)
docker compose up core               # free-proxy mode: run core alone, no vault
```

Health probes: `curl localhost:8080/health` (core), `curl localhost:3000/health` (vault, exposed on host port `3010`).

### Vault development (cd vault/)

```bash
npm install
npm run db:migrate                   # apply drizzle migrations
npm run dev                          # tsx watch, WALLET_BACKEND=simulated by default
npm test                             # vitest run (uses in-memory SQLite, test tokens)
npm run test:watch
npm run db:generate                  # regenerate migrations after src/db/schema.ts changes
```

Run a single test file or pattern:
```bash
npx vitest run tests/integration/lightning.test.ts
npx vitest run -t "reserve releases on failure"
```

Regtest Lightning + Cashu environment for `WALLET_BACKEND=lightning|auto`:
```bash
docker compose -f docker-compose.dev.yml up -d     # bitcoind + lnd-alice + lnd-bob + nutshell
# Manual channel setup steps are commented at the top of docker-compose.dev.yml
```

## Service boundaries and data flow

```
client → core:8080 ──(reserve)──► vault:3000 ──► LND / Nutshell mint
                    ◄─(settle/release after response)
         └─► provider (mesh-llm / Routstr / any OpenAI-compatible URL)
```

1. Request hits **core** (OpenAI-compatible proxy, port 8080).
2. Core calls **vault** `/internal/reserve` to debit buyer's agent balance (default 4096 tokens — see `[vault]` in `config.toml`).
3. Core picks cheapest qualified provider from `config.toml` `[[providers]]`.
4. Response streams back; core calls `/internal/settle` (actual cost) or `/internal/release` (unused).
5. Seller payout goes back through vault to Lightning.

**Trust boundary:** core ↔ vault is a shared-secret `X-Internal-Token` (`VAULT_INTERNAL_TOKEN`, constant-time compare). Admin routes use a separate bearer token (`VAULT_ADMIN_TOKEN`). Agent routes use per-agent bearer tokens returned at registration.

**Service dependencies in `docker-compose.yml`:** `lnd` + `mint` must be healthy → `vault` starts → `vault` healthy → `core` starts. Edits to `config.toml` only affect core (bind-mounted read-only), so `docker compose restart core` is the right cycle.

## Vault architecture (./vault/src/)

**Factory pattern with dependency injection** — this is load-bearing, don't bypass it:

- `app.ts` → `buildApp({ db?, wallet?, cashuWallet?, loggerStream? })` builds the Fastify instance. DB and wallet are decorated onto `app` (`app.db`, `app.paymentsService`). Routes **must** read `app.paymentsService`, never import it directly. Tests use in-memory SQLite + simulated wallet through this same factory.
- `index.ts` is the production entrypoint: runs migrations, dynamically imports lightning/cashu backends (lazy — avoids loading unused packages), calls `buildApp`, starts a 30s interval for approval timeouts. **Never connect to LND inside `buildApp`** — do it in `index.ts` and pass the wallet in. Test runs should never touch that interval.

**Ledger pattern — RESERVE / RELEASE / PAYMENT entries.** Async wallet calls (LN) can crash between "decided to pay" and "paid." The ledger writes a `RESERVE` before the wallet call and a `PAYMENT` (on success) or `RELEASE` (on failure/release) after. Balance = sum of all entries. This is how crash recovery works — **do not "simplify" by skipping the reserve step.**

**Policy engine is append-only and versioned.** `PATCH /operator/agents/:id/policy` creates a new row in `policy_versions`; historical decisions reference the version that was in force at the time. `fail-closed DENY on error` — if the engine throws, deny. Don't add escape hatches.

**Wallet routing** (`modules/payments/payments.service.ts::selectRail`): below `CASHU_THRESHOLD_MSAT` → Cashu (instant, zero fee). At/above → Lightning (final settlement). Agent `preferred_rail` hint in a payment request overrides the threshold. Falls back to simulated if neither real rail is configured.

**Internal billing routes (`/internal/reserve|settle|release`) are only registered when `VAULT_INTERNAL_TOKEN` is set** — see the conditional `app.register(internalBillingRoutes)` in `app.ts`. Omitting the env var is the feature flag.

**Database**: SQLite + Drizzle ORM, WAL mode, `foreign_keys=ON`, `synchronous=NORMAL`. Schema in `src/db/schema.ts`; migrations generated into `src/db/migrations/` — never hand-edit migrations. In tests, `DATABASE_PATH=:memory:` (set in `vitest.config.ts`).

**Validation**: Zod v4 schemas via `fastify-type-provider-zod`. Config is parsed once in `src/config.ts` with cross-field refinements (LND vars required iff `WALLET_BACKEND` is `lightning`/`auto`; similar for Cashu). Import-time validation — misconfiguration is caught at startup, not request time.

## Planning artifacts

`vault/.planning/` contains the GSD workflow artifacts (milestones, phases, plans, research). These are history and design context — they are **not** live docs. Prefer reading the current code over planning files for architectural questions, but check `vault/.planning/research/ARCHITECTURE.md` and phase `SUMMARY.md` files for the "why" behind non-obvious decisions.
