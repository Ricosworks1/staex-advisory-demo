# Data Architecture for Regulated Financial Institutions

## The core principle

**Client data never leaves Bitcoin Suisse's infrastructure.**

The advisory agents run inside Bitcoin Suisse's own regulatory perimeter. The only data that crosses the boundary is the BPF intelligence feed — encrypted, signed, and delivered over MCC. No client name, no account number, no position size, no PII ever reaches an external system.

---

## The problem with standard AI deployments

Most institutions deploying AI send client context directly to an LLM API:

```
Client portfolio → Claude API (Anthropic servers, US)
```

This creates four compliance failures for a regulated institution:

1. **Data residency violation** — client data leaves Swiss/EU/Bermuda jurisdiction
2. **No audit trail** — no cryptographic proof of what data was sent and when
3. **PII exposure** — client identity travels with portfolio context
4. **Training risk** — without a DPA, data may be used to train future models

Bitcoin Suisse, operating under FINMA (Switzerland), BMA (Bermuda), and pursuing MiCAR (Liechtenstein/EU), cannot operate this way.

---

## The Staex architecture

Three deployment tiers. Bitcoin Suisse chooses based on their risk appetite and IT capability.

---

### Tier 1 — Fully self-hosted (recommended for FINMA/BMA compliance)

```
┌─────────────────────────────────────────────────────────────┐
│                  Bitcoin Suisse Infrastructure               │
│                                                              │
│  ┌──────────────┐    ┌────────────────────────────────────┐  │
│  │ Client data  │    │       Advisory Agent Cluster       │  │
│  │  (encrypted  │───▶│  CIO · PM · Risk · Quant · RM     │  │
│  │  at rest)    │    │  (running on BS servers)           │  │
│  └──────────────┘    └────────────┬───────────────────────┘  │
│                                   │ anonymised query only     │
│                      ┌────────────▼───────────────────────┐  │
│                      │     Anonymisation Layer            │  │
│                      │  strips: name, account, exact $    │  │
│                      │  keeps: profile type, % allocation │  │
│                      └────────────┬───────────────────────┘  │
│                                   │                           │
│  ┌────────────────────────────────▼───────────────────────┐  │
│  │                    MCC Node                            │  │
│  │  Ed25519-signed · ChaCha20-Poly1305 encrypted         │  │
│  └─────────────────────────────┬──────────────────────────┘  │
└────────────────────────────────│────────────────────────────┘
                                 │ MCC tunnel (encrypted)
┌────────────────────────────────▼────────────────────────────┐
│                    Staex Infrastructure                      │
│                                                              │
│  BPF Intelligence Feed:                                      │
│  - Monthly thesis memos                                      │
│  - CIO strategy signals                                      │
│  - Quant backtest updates                                    │
│  - Chain-level on-chain data                                 │
│                                                              │
│  Claude API (Anthropic) — called by BS agents               │
│  with anonymised context only                               │
└──────────────────────────────────────────────────────────────┘
```

**What stays inside Bitcoin Suisse:** everything with client identity, account numbers, exact portfolio values, transaction history, KYC data.

**What crosses via MCC:** BPF intelligence updates (thesis memos, chain data, strategy signals). This is research, not client data. It flows one way: Staex → Bitcoin Suisse.

**What goes to Claude API:** anonymised portfolio context only.
```
❌ "Livio Bühler holds 2.4 BTC worth CHF 154,000"
✅ "Profile 5 client. 60% BTC, 25% HYPE, 15% USDC. Question: rebalancing trigger?"
```

---

### Tier 2 — Hybrid (fastest to deploy, suitable for Bermuda entity)

```
Bitcoin Suisse client interface (their servers)
         ↓ anonymised query
Staex advisory proxy (our servers, EU)
         ↓ anonymised context
Claude API (Anthropic, with DPA)
         ↓
Staex advisory proxy
         ↓ recommendation
Bitcoin Suisse (displayed to RM or client)
```

Client data is anonymised before leaving Bitcoin Suisse servers. Staex acts as the processing intermediary. Suitable for the Bermuda advisory entity where FINMA rules don't apply, and BMA's requirements can be met with appropriate DPAs and contractual protections.

**This is what the current demo runs.** It demonstrates the interface — not the production data architecture.

---

### Tier 3 — Staex-hosted demo (current, for evaluation only)

The demo at `staex.gitlab.io/pm/staex-advisory-crypto-portfolio-management` runs with mock data. No real client data exists. This is the proof of concept — not the production deployment.

---

## Token optimisation — the compliance dividend

This architecture directly addresses both items sold to Livio Bühler:

### 1. Token cost reduction

The anonymisation layer is also a token efficiency layer:

| Without anonymisation | With anonymisation |
|----------------------|-------------------|
| Full client record sent with every query | Only profile type + anonymised allocation |
| Repeated context in every message | Prompt caching on the system prompt (static) |
| No caching possible (PII changes caching keys) | 90%+ cache hit rate on system prompts |
| ~4,000 tokens per advisory query | ~800 tokens per advisory query |
| **Cost: ~$0.012 per query** | **Cost: ~$0.002 per query** |

**80% token cost reduction** is achievable through proper anonymisation + prompt caching — without reducing advisory quality.

This is the token optimisation audit. Bitcoin Suisse's current AI deployments almost certainly send full context with every query. Staex shows them how to redesign the prompt architecture to achieve 80% savings.

### 2. Bermuda advisory architecture

Tier 1 or Tier 2 above is the Bermuda architecture. The agents run inside the regulatory perimeter. The intelligence (BPF) comes in via MCC. Client outputs are reviewed by a licensed RM before delivery. This satisfies BMA's Class F DABA requirements for discretionary portfolio management.

---

## The MCC audit trail — why it matters for compliance

Every MCC data transmission is:
- **Ed25519 signed** at source — cryptographic proof of origin, tamper-evident
- **ChaCha20-Poly1305 encrypted** — unreadable in transit
- **Logged with timestamp** — immutable sequence of events

For a regulated institution, this means:

> "Every piece of investment intelligence that entered our system was cryptographically signed and timestamped. We can prove, to the day and second, what data our agents received and when."

This replaces the manual compliance documentation that Livio's team currently generates. The audit trail is automatic, cryptographically unassailable, and regulatorily robust across FINMA, BMA, MiCAR, and FATF frameworks.

**The pitch to Livio:** *"Your compliance team spends time generating paper that proves data wasn't tampered with. MCC makes that proof automatic. Every intelligence update that reaches your advisory agents is self-evidently auditable — no manual report needed."*

---

## Data classification

| Data type | Classification | Stays in BS? | Crosses MCC? | Goes to Claude? |
|-----------|---------------|-------------|-------------|----------------|
| Client name / identity | PII — Confidential | ✅ Always | ❌ Never | ❌ Never |
| Account numbers | PII — Confidential | ✅ Always | ❌ Never | ❌ Never |
| Exact portfolio $ values | Financial — Confidential | ✅ Always | ❌ Never | ❌ Never |
| Profile type (1–10) | Anonymised | ✅ Stored | ❌ Never | ✅ Yes |
| % allocation by asset | Anonymised | ✅ Stored | ❌ Never | ✅ Yes |
| BPF thesis memos | Research — Public | Received | ✅ Inbound | ✅ Yes |
| Agent recommendations | Advisory output | ✅ Stored | ❌ Never | Reference only |
| Compliance audit logs | Regulatory | ✅ Always | ❌ Never | ❌ Never |

---

## Regulatory mapping

| Regulation | Requirement | How this architecture satisfies it |
|-----------|-------------|-----------------------------------|
| **FINMA** | Data residency in Switzerland for Swiss clients | Tier 1: all data on BS Swiss servers |
| **BMA (Bermuda)** | Documented advisory process, client suitability | Agent outputs logged with timestamp, profile assignment by principal |
| **MiCAR (EU/Liechtenstein)** | Data protection, operational resilience | Anonymisation layer + MCC encrypted transport |
| **GDPR** | Right to erasure, data minimisation | Only anonymised data leaves BS perimeter |
| **FATF** | AML/KYC audit trail | MCC-signed logs, agent decision records |
| **NIS2 (EU)** | Network security for financial infrastructure | MCC replaces open internet transport |

---

## What Bitcoin Suisse needs to deploy Tier 1

| Component | Provided by | Effort |
|-----------|------------|--------|
| MCC node installation on BS servers | Staex (1 day) | Low |
| Agent cluster deployment (Docker) | Staex (1 week) | Medium |
| Anonymisation layer integration with BS client DB | Staex + BS IT team | Medium |
| Anthropic enterprise DPA (if using Claude API) | Bitcoin Suisse legal | Low |
| BPF intelligence subscription | Maze2 SA licence | Contract only |
| RM workflow integration | Staex (2 weeks) | Medium |

**Total deployment estimate: 4–6 weeks for a production-ready Tier 1 system.**

---

## What this is not

- This is not a trading system. Agents generate recommendations. A licensed human principal approves and executes.
- This is not a black box. Every agent prompt is documented in this repository. Bitcoin Suisse audits the system prompts at any time.
- This is not vendor lock-in. The static client interface is open HTML. The agent prompts are open markdown. If Bitcoin Suisse wants to migrate, they take their data and the prompts with them.
