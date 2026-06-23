# CLAUDE.md — AI Agent Context

Read this before doing any work in this repository. This file is the authoritative context for any AI agent operating in this project.

---

## What this project is

An AI-powered crypto investment advisory system for UHNWI and family offices. It operates under the Bitcoin Suisse International regulatory umbrella (Bermuda: Class F DABA + Class B IBA). The advisory output is human-reviewed before client delivery — this system is the intelligence layer, not the decision layer.

## Who built this and why

- **Staex GmbH** (Berlin) — AI and secure networking infrastructure. Ricardo Mastrangelo, CGO.
- **Bitcoin Suisse International Ltd** (Bermuda) — the regulated entity under which advisory services are delivered.
- **Maze2 SA** — publisher of the Blockchain Payment Flow (BPF) analysis, the intelligence source for all theses.

The system exists because UHNWI crypto clients at Bitcoin Suisse Bermuda have no daily AI-powered advisory. Their portfolio sits in custody. Nobody actively manages it or communicates thesis-level thinking to them. This system fills that gap.

## The investment philosophy (read THESIS.md for full detail)

Four axioms drive every decision:
1. All crypto tokens are options on underlying ideas — most expire zero, a few return 10x–100x
2. Crypto is winner-takes-all within each category
3. Bitcoin is a religion (store of value, not technology bet)
4. Tsunamis can't be predicted, only positioned for — the BPF is the seismograph

## The intelligence source

**Blockchain Payment Flow Analysis** — `github.com/Ricosworks1/blockchain-payment-flow-analysis`
- On-chain economic research covering 25+ L1/L2 networks and 20+ protocols
- Published quarterly by Maze2 SA
- Data sourced from DeFiLlama, CoinGecko, Mempool.space, Token Terminal
- Snapshots in `data/bpf/` — always use the most recent unless doing historical analysis

The Oct 2025 BPF snapshot was the first publication. It identified HYPE (Hyperliquid) as structurally differentiated: $13B annualized revenue, 97% buyback model, 70% DEX market share. HYPE went from $50 (Oct 2025) to $77 ATH (Jun 2026) while BTC dropped 49% from its Oct ATH. This validated the thesis and is the canonical backtest example.

## The 10 investment profiles

Defined in `profiles/`. Each profile maps to an option-book analogy (see THESIS.md). Profile 1 is max aggressive (deep OTM call book). Profile 10 is max conservative (pure stablecoin yield). Client assignment to a profile is done by the human principal — agents do not assign profiles.

## The agent team

Six agents, each with a single mandate. Defined in `agents/` and `AGENTS.md`.

Execution order for a new BPF report:
1. Research Analyst → thesis memo
2. CIO → monthly strategy
3. Portfolio Manager → position delta
4. Risk Manager → validation
5. Quant → backtest signal
6. Relationship Manager → client briefing

All agents run with Claude Sonnet 4.6 via `CLAUDE_CODE_OAUTH_TOKEN`. Never use `ANTHROPIC_API_KEY` — that is for paying customers only.

## What agents must not do

- Predict prices
- Promise returns
- Assign clients to profiles
- Execute trades
- Send client communications without principal review
- Modify hard risk limits
- Override the Risk Manager's hard limits

## Public demo URL and deployment

**Live URL: `https://staex.io/advisory`** — publicly accessible, no login required.

This is the canonical URL to share with Andrej Majcen, Livio Bühler, or any external party.

### How it works (no manual steps needed)
- The demo HTML source is `public/index.html` in this repo
- `staex.io/advisory` is a Next.js route in `~/projects/staex.io-neo` that proxies the raw file from this GitLab repo via the GitLab API
- **Just push to this repo → staex.io/advisory updates within 60 seconds automatically**
- No sync scripts, no second commit, no gymnastics

### What NOT to do
- Do not run `sync-to-staex.sh` — it's obsolete and can be deleted
- Do not edit `~/projects/staex.io-neo/public/advisory/index.html` — it's a stale copy, ignored
- Do not commit the HTML to staex.io-neo — the proxy reads it live from this repo

### If the demo page needs updating
1. Edit `public/index.html` in this repo
2. `git add public/index.html && git commit -m "..." && git push`
3. Done — staex.io/advisory shows the update within ~60s

## Key files

| File | Purpose |
|------|---------|
| `THESIS.md` | The axioms. Read before forming any opinion. |
| `AGENTS.md` | Team structure, mandates, interaction protocol |
| `profiles/` | 10 investment profiles with allocation specs |
| `agents/*/prompt.md` | System prompt for each agent |
| `data/bpf/` | BPF report snapshots — the intelligence input |
| `backtests/` | Historical thesis performance — learn from these |
| `public/index.html` | The live demo page — source of truth for staex.io/advisory |

## Tone and communication standards

- Never use jargon without defining it
- Never cite past performance without the invalidation condition
- Every recommendation must state: the thesis, the structural condition, and what would prove it wrong
- Client-facing content (Relationship Manager output only) must be readable by a non-technical UHNWI. No formulas, no raw data dumps.
- Internal memos (all other agents) are structured markdown — headers, tables, bullet points. No prose paragraphs.

## Regulatory context

All advisory outputs are delivered under Bitcoin Suisse International Ltd. The Bermuda Monetary Authority requires:
- Clear separation of research and advice
- Documented rationale for every recommendation
- Client suitability documentation (maintained by the human principal, not agents)
- No leveraged recommendations without explicit principal approval

Every agent output that references a position or allocation must include: the BPF evidence base, the thesis, and the invalidation condition. This is not optional — it is the audit trail.
