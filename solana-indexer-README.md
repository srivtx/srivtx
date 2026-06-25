<img src="https://raw.githubusercontent.com/srivtx/solana-indexer-skill/main/docs/assets/indexer.webp" alt="solana-indexer" width="100%" />

# solana-indexer

> The first skill that teaches you how to **build** a Solana indexer from scratch.
> Fills the `indexing-geyser` gap in the Solana AI Kit Ideas table.

[![MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![TS](https://img.shields.io/badge/TypeScript-strict-blue.svg)](https://www.typescriptlang.org)
[![Rust](https://img.shields.io/badge/Rust-stable-orange.svg)](https://www.rust-lang.org)
[![Claude Code](https://img.shields.io/badge/Claude_Code-skill-d97757.svg)](https://docs.anthropic.com/en/docs/claude-code)
[![Bounty](https://img.shields.io/badge/Superteam_Bounty-PR_%2349-14F195.svg)](https://github.com/solanabr/skill-bounty/pull/49)

---

## What is this?

Every serious Solana dApp — DeFi dashboards, NFT marketplaces, gaming leaderboards, social graphs, analytics — needs a **custom indexer**. The AI Kit had Helius, QuickNode, and solana-agent-kit skills for *consuming* indexed data. Nothing for *building* one.

This skill is the missing piece. It routes Claude Code (or Codex) to the right reference based on intent, with working code in two languages.

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/srivtx/solana-indexer-skill/main/install.sh | bash
```

Installs to `~/.claude/skills/solana-indexer/` (and `~/.codex/skills/solana-indexer/` if Codex is installed).

## What you get

| | |
|---|---|
| **9 reference docs** | indexer-architecture, geyser-plugins, postgres-schemas, backfill-strategies, real-time-streaming, cost-optimization, testing-indexers, production-ops, resources |
| **3 runnable examples** | `minimal-indexer-ts` (Helius webhook → Postgres, `tsc --noEmit` clean), `geyser-plugin/skeleton` (Rust, `cargo check` clean), `subgraph-template` (The Graph on Solana) |
| **2 agents** | `indexer-architect` (designs an indexer for a dApp), `indexer-qa` (tests for correctness, dedup, replay safety) |
| **2 commands** | `/build-indexer <spec>` (scaffolds a complete indexer), `/backfill [range]` (backfills with checkpointing) |
| **1 rule** | `indexer-defaults` (auto-loads in `/indexer` folders) |

## The architecture (5 ingestion methods, 1 skill)

```
                ┌──────────────────────────────────┐
                │  SKILL.md (3.9 KB routing hub)    │
                └──────────────┬───────────────────┘
                               │
        ┌──────────────┬───────┴───────┬──────────────┐
        │              │               │              │
   Geyser gRPC   WebSocket      Webhooks        Polling       Subgraph
   Rust / <100ms TS / 200-500ms any / 1-5s     any / 5-30s    GraphQL
        │              │               │              │
        └──────────────┴───────┬───────┴──────────────┘
                               ▼
                  ┌─────────────────────────┐
                  │  your indexer (the skill) │
                  └──────────┬──────────────┘
                             ▼
                ┌────────────────────────┐
                │  Postgres 16             │
                │  NUMERIC(40,0) · BYTEA(32) │
                └──────────┬──────────────┘
                           ▼
                  ┌────────────────────┐
                  │  your dApp         │
                  └────────────────────┘
```

## How it works

You ask a question. The skill routes you to the right reference.

```
You say:                                  The skill reads:
────────────────────────────────────────  ─────────────────────────────────
"I need to build an indexer for X"        references/indexer-architecture.md
"Geyser / Yellowstone gRPC"               references/geyser-plugins.md
"Postgres schema for [swaps/NFTs/pools]"  references/postgres-schemas.md
"How do I backfill [N days]"              references/backfill-strategies.md
"Real-time WebSocket / gRPC patterns"     references/real-time-streaming.md
"Reduce RPC costs"                        references/cost-optimization.md
"Test my indexer"                         references/testing-indexers.md
"Run in production"                       references/production-ops.md
"Where are the official docs"             references/resources.md
```

`SKILL.md` is a **3.9 KB routing hub**. The nine reference docs load **only** when the agent needs them. Idle context cost: **<4 KB**.

## The examples (real working code, not markdown)

### `examples/minimal-indexer-ts/`

100-line TypeScript indexer using Helius webhooks → Postgres. `tsc --noEmit` clean. Runnable against devnet.

```ts
import { helius, webhook } from "helius-sdk";
import { Client } from "pg";

const client = new Client("postgres://localhost/indexer");
const wh = webhook(async (tx) => {
  await client.query(
    "INSERT INTO swaps (slot, sig) VALUES ($1, $2)",
    [tx.slot, tx.signature]
  );
});

helius("YOUR_API_KEY").on("enhancedTransactions", wh);
```

### `examples/geyser-plugin/skeleton/`

Rust Geyser plugin skeleton using `solana-geyser-plugin-interface 1.18`. `cargo check` clean. Filter `ProgramSubscribe` → batched Postgres writes.

### `examples/subgraph-template/`

The Graph on Solana: manifest, schema, mapping stub. Compressed subgraph.

## Verification

| Check | Result |
|---|---|
| TypeScript example typecheck | `tsc --noEmit` **PASS** |
| Rust example typecheck | `cargo check` **PASS** |
| SKILL.md link integrity | **9/9 references resolve** |
| Postgres schemas verified | vs Raydium CLMM `PoolState` source |
| All 9 refs cross-checked | vs real SDK + official docs |
| Standalone repo CI | **6 jobs passing** |
| Dedup key patterns verified | `(slot, signature)` and `(pubkey, slot)` |

## Real-world indexers this skill helps you build

- Raydium CLMM swap indexer (Yellowstone gRPC → Postgres, ~50K swaps/min on mainnet)
- Magic Eden listing indexer (Helius webhooks → Postgres, ~100K listings/day)
- Jupiter aggregator route indexer (Helius enhanced tx → Postgres, ~10M routes/day)
- Drift perp position tracker (Helius account sub → Postgres, ~1K liquidations/day)
- Meteora DLMM bin indexer (Geyser program sub → TimescaleDB, ~5K swaps/min)
- Token-2022 transfer-hook indexer (Helius enhanced tx → Postgres)
- Governance proposal tracker (Geyser program sub → Postgres)

## Composes with

This skill fills the **build** layer. It composes with:

- `ext/helius` — for *using* Helius APIs inside the indexer
- `ext/sendai/skills/quicknode` — for QuickNode streams
- `ext/solana-dev` — for general Solana program patterns
- `ext/cloudflare` — for deploying the indexer on Workers
- `ext/trailofbits` — if the indexer handles sensitive data

## Bounty

Entered in the **Superteam Brasil "Ship useful agent skills"** bounty as PR [#49](https://github.com/solanabr/skill-bounty/pull/49). Winner announced **July 8, 2026**.

## License

[MIT](LICENSE) — Built by [@srivtx](https://github.com/srivtx).
