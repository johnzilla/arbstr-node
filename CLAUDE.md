# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository shape

This is a **meta-repo** that composes the full arbstr stack (sats-metered inference routing) from four services. **No service source lives here** — it is orchestration only, and every service runs from a published image pinned to a release:

- `./` (root) — `docker-compose.yml`, `config.toml` (arbstr core config), `.env.example`.
- **core** — `ghcr.io/johnzilla/arbstr:<version>`. Rust routing engine; source and releases in [`johnzilla/arbstr`](https://github.com/johnzilla/arbstr).
- **vault** — `ghcr.io/johnzilla/arbstr-vault:<version>`. Node.js/TypeScript/Fastify treasury; source and releases in [`johnzilla/arbstr-vault`](https://github.com/johnzilla/arbstr-vault). **Vault code changes go to that repo**, not this one.
- **lnd** / **mint** — third-party images (`lightninglabs/lnd`, `cashubtc/nutshell`).

The `vault` directory that contains this checkout's parent (`~/Desktop/vault/projects/`) is unrelated — it's the user's personal projects folder.

### Versions and releases

Each arbstr-node release pins exact image versions in `docker-compose.yml`, and its GitHub Release notes list them. Core and vault version independently (semver, signed tags, GitHub Releases with build provenance — see their DEVELOPMENT.md / README). To move the stack to a new core or vault release:

```bash
# 1. Edit the image tag in docker-compose.yml (Dependabot also proposes these weekly)
# 2. Deploy and verify
docker compose pull && docker compose up -d
# 3. Commit; cut an arbstr-node release (signed tag vX.Y.Z) when the set is worth shipping
```

Verify a pinned image's provenance: `gh attestation verify oci://ghcr.io/johnzilla/arbstr:<version> --owner johnzilla`.

## Common commands

### Full stack (root)

```bash
cp .env.example .env                 # set VAULT_ADMIN_TOKEN and VAULT_INTERNAL_TOKEN (both ≥32 chars)
docker compose up -d                 # start core, vault, lnd, mint
docker compose logs -f core          # follow core logs
docker compose up -d --force-recreate core   # pick up config.toml edits (see below)
docker compose up -d --no-deps core  # free-proxy mode (after removing [vault] from config.toml)
```

Health probes: `curl localhost:8080/health` (core), `curl localhost:3010/health` (vault — compose remaps host 3010 → container 3000).

### Vault development

Happens in a checkout of [`arbstr-vault`](https://github.com/johnzilla/arbstr-vault), not here (Node 22, `npm test`, `npm run dev`; see its README). To try an unreleased vault in this stack, point `vault.image` at its `:<short-sha>` tag (published on every merge to its main) and switch back to a release before committing.

## Service boundaries and data flow

```
client → core:8080 ──(reserve)──► vault:3000 ──► LND / Nutshell mint
                    ◄─(settle/release after response)
         └─► provider (mesh-llm / Routstr / any OpenAI-compatible URL)
```

1. Request hits **core** (OpenAI-compatible proxy, port 8080) with the agent's bearer token.
2. Core calls **vault** `/internal/reserve` with an estimate at frontier rates (4096 output tokens when the client sends no `max_tokens` — see `[vault]` in `config.toml`). Vault applies the agent's policy (per-request max, daily limit) and returns 403 with a `reason` on denial. New agents have a deny-all policy until `PATCH /operator/agents/:id/policy` sets limits.
3. Core picks cheapest qualified **routable** provider from static `[[providers]]` and/or live `[tokenstats]` quotes (rates in **sats per 1M** tokens). Market sources need keys in `[tokenstats.keys]`; see `GET /providers` → `market.skipped`.
4. Response streams back; core calls `/internal/settle` (actual cost) or `/internal/release` (unused).
5. Vault releases any reservation core never settled once it is older than `BILLING_RESERVATION_TTL_MS` (15 min).

Seller payout is not built — vault only meters buyer spend.

**Trust boundary:** core ↔ vault is a shared-secret `X-Internal-Token` (`VAULT_INTERNAL_TOKEN`, constant-time compare). Admin routes use a separate bearer token (`VAULT_ADMIN_TOKEN`). Agent routes use per-agent bearer tokens returned at registration.

**Service dependencies in `docker-compose.yml`:** `lnd` + `mint` must be healthy → `vault` starts → `vault` healthy → `core` starts. Edits to `config.toml` only affect core. It is a single-file bind mount, so use `docker compose up -d --force-recreate core`: `restart` keeps the old file if the editor replaced it (as `sed -i` and many editors do).

## Vault architecture (arbstr-vault `src/`)

**Factory pattern with dependency injection** — this is load-bearing, don't bypass it:

- `app.ts` → `buildApp({ db?, wallet?, cashuWallet?, loggerStream? })` builds the Fastify instance. DB and wallet are decorated onto `app` (`app.db`, `app.paymentsService`). Routes **must** read `app.paymentsService`, never import it directly. Tests use in-memory SQLite + simulated wallet through this same factory.
- `index.ts` is the production entrypoint: runs migrations, dynamically imports lightning/cashu backends (lazy — avoids loading unused packages), calls `buildApp`, starts a 30s interval for approval timeouts and a 60s sweeper for stale billing reservations. **Never connect to LND inside `buildApp`** — do it in `index.ts` and pass the wallet in. Test runs should never touch that interval.

**Ledger pattern — RESERVE / RELEASE / PAYMENT entries.** Async wallet calls (LN) can crash between "decided to pay" and "paid." The ledger writes a `RESERVE` before the wallet call and a `PAYMENT` (on success) or `RELEASE` (on failure/release) after. Balance = sum of all entries. This is how crash recovery works — **do not "simplify" by skipping the reserve step.**

**Policy engine is append-only and versioned.** `PATCH /operator/agents/:id/policy` creates a new row in `policy_versions`; historical decisions reference the version that was in force at the time. `fail-closed DENY on error` — if the engine throws, deny. Don't add escape hatches.

**Wallet routing** (`modules/payments/payments.service.ts::selectRail`): below `CASHU_THRESHOLD_MSAT` → Cashu (instant, zero fee). At/above → Lightning (final settlement). Agent `preferred_rail` hint in a payment request overrides the threshold. Falls back to simulated if neither real rail is configured.

**Internal billing routes (`/internal/reserve|settle|release`) are only registered when `VAULT_INTERNAL_TOKEN` is set** — see the conditional `app.register(internalBillingRoutes)` in `app.ts`. Omitting the env var is the feature flag.

**Database**: SQLite + Drizzle ORM, WAL mode, `foreign_keys=ON`, `synchronous=NORMAL`. Schema in `src/db/schema.ts`; migrations generated into `src/db/migrations/` — never hand-edit migrations. In tests, `DATABASE_PATH=:memory:` (set in `vitest.config.ts`).

**Validation**: Zod v4 schemas via `fastify-type-provider-zod`. Config is parsed once in `src/config.ts` with cross-field refinements (LND vars required iff `WALLET_BACKEND` is `lightning`/`auto`; similar for Cashu). Import-time validation — misconfiguration is caught at startup, not request time.

**Node version**: vault needs Node 22 (the Docker image uses `node:22-slim`); `better-sqlite3` does not build on newer majors.
