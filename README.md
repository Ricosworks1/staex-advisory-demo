# Staex Advisory — Crypto Portfolio Management

AI-powered investment advisory for UHNWI and family offices. Regulated under Bitcoin Suisse International Ltd (Bermuda: Class F DABA + Class B IBA, issued May 12, 2026).

**Core idea:** A multi-agent investment team grounded in the [Blockchain Payment Flow Analysis](https://github.com/Ricosworks1/blockchain-payment-flow-analysis) engine. Agents think, argue, and produce — a human principal decides.

---

## Structure

```
CLAUDE.md                   — read first. full context for any AI agent entering this repo
THESIS.md                   — the investment axioms. the "why" behind every decision
AGENTS.md                   — who does what, how agents interact, escalation protocol
profiles/                   — 10 investment profiles (aggressive → conservative)
agents/                     — system prompts for each agent role
  cio/                      — Chief Investment Officer
  portfolio_manager/        — position sizing, rebalancing, execution
  risk_manager/             — drawdown, VaR, stop-loss, concentration
  quant/                    — statistical analysis, backtesting, signal generation
  research/                 — BPF report reading, thesis generation
  relationship_manager/     — client-facing, daily briefings, reports
data/bpf/                   — BPF report snapshots (intelligence input)
data/benchmarks/            — historical price data for backtesting
data/live/                  — live on-chain data feeds
backtests/                  — documented runs with P&L, methodology, lessons
reports/client/             — generated client-facing advisory reports
```

## Intelligence source

All theses are grounded in the Blockchain Payment Flow Analysis — proprietary on-chain economic research covering 25+ networks and 20+ protocols, published quarterly by Maze2 SA. Licence required for commercial use.

## Not financial advice

This system is a research and advisory intelligence layer. All outputs require review by a licensed principal before client delivery. Operated under BMA regulatory framework.
