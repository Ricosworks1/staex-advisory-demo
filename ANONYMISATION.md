# Anonymisation & Prompt Caching — Technical Specification

This document describes the two mechanisms that make the advisory system compliant for regulated financial institutions and cost-efficient at scale. Both are implemented and verifiable in the source code.

---

## 1. PII Anonymisation Middleware

**File:** `src/app/api/advisory/lib/anonymise.ts` (staex.io-neo repo)

Every message from a client, RM, or internal user passes through this layer before any call to an external LLM API. The layer is server-side — it runs inside the regulated institution's infrastructure perimeter.

### What it strips

| Pattern | Example input | Output |
|---------|--------------|--------|
| Email addresses | `contact@bitcoinsuisse.com` | `[EMAIL REDACTED]` |
| Phone numbers | `+41 78 734 80 82` | `[PHONE REDACTED]` |
| IBAN / bank accounts | `CH56 0483 5012 3456 7800 9` | `[IBAN REDACTED]` |
| Swiss AHV (SSN) | `756.1234.5678.90` | `[SSN REDACTED]` |
| Passport / ID numbers | `X1234567` | `[ID REDACTED]` |
| Titled names | `Dr. Andrej Majcen` | `[PERSON REDACTED]` |
| Known names (operator list) | `Livio Bühler` | `[PERSON REDACTED]` |
| Large currency amounts | `CHF 1,500,000` | `[LARGE AMOUNT]` |
| Mid-range amounts | `USD 250,000` | `[MID-RANGE AMOUNT]` |
| Small amounts (token prices) | `$64,000` | `$64,000` (kept) |

### What it preserves

- Crypto asset names (BTC, HYPE, ETH, USDC)
- Percentage allocations (60%, 25%, 15%)
- Market prices (kept — they are not PII)
- Small amounts under $10,000
- Profile numbers and anonymised portfolio structure

### What Claude receives

Before anonymisation:
```
My client Dr. Livio Bühler holds CHF 2,400,000 in BTC. His IBAN is CH56 0483 5012 3456 7800 9.
Should he rebalance given current market conditions?
```

After anonymisation:
```
My client [PERSON REDACTED] holds [LARGE AMOUNT] in BTC.
His IBAN is [IBAN REDACTED].
Should he rebalance given current market conditions?
```

Claude receives enough context to answer the advisory question. It never receives the client's identity or exact financial position.

### Audit logging

Every anonymisation event is logged server-side with:
- Request ID (UUID, not tied to client identity)
- Types of PII detected (not the original values)
- Count per type

Log format:
```
[advisory-anonymise] reqId=550e8400-e29b-41d4-a716-446655440000 pii_detected=true redactions={"email":1,"currency_amount":2}
```

Original PII values are never written to logs.

### Operator configuration

The `KNOWN_NAMES` list in `anonymise.ts` is empty by default. The deploying institution populates it with their staff and client names. In production, this list is loaded from encrypted configuration — not hardcoded in source.

### Limitations (documented honestly)

- Name detection without a title (`Dr.`, `Mr.`, etc.) is not guaranteed. Bare "First Last" patterns that don't match known names may pass through.
- Regex-based detection is a compliance layer, not a security boundary. It complements access controls — it does not replace them.
- Amounts under $10,000 are kept. A client mentioning a $9,500 position will have that number passed to Claude. This is a design choice: stripping all numbers would make the advisory responses useless.

---

## 2. Prompt Caching

**File:** `src/app/api/advisory/chat/route.ts` (staex.io-neo repo)

### The problem with naive LLM deployments

Most advisory systems send the full context — investment thesis, portfolio state, market data, client profile — with every message. This means:

- Every query pays for 1,800+ tokens of static context
- No caching is possible because context changes slightly each time
- Cost compounds with every client, every query, every day

### The caching architecture

The system prompt is split into two blocks:

```
Block 1 — STATIC_CONTEXT (~1,800 tokens) ← cached for 5 minutes
├── Investment thesis (6 axioms)
├── BPF October 2025 intelligence snapshot
├── 10 investment profiles
├── Current market state
└── Communication standards

Block 2 — Agent mandate (~80 tokens) ← sent fresh each call
└── The specific role this agent plays (CIO / RM / Risk / Quant / etc.)
```

Anthropic caches Block 1 across all calls made within a 5-minute window. After the first call, all subsequent calls in that window pay only for:
- Block 2 (80 tokens, uncached)
- The user message (variable, anonymised)
- Output tokens

### Cost comparison

| Scenario | Input tokens per call | Cost per call (Sonnet) |
|----------|----------------------|----------------------|
| No caching (naive) | ~1,900 tokens | ~$0.006 |
| First call (cache creation) | 1,900 tokens + cache write | ~$0.008 |
| Subsequent calls (cache hit) | ~160 tokens + cache read | ~$0.0015 |
| **Saving per cached call** | | **~75%** |

At 1,000 advisory queries per day across a UHNWI book of 50 clients:
- Without caching: ~$6/day → $2,190/year
- With caching (90% cache hit rate): ~$0.90/day → $329/year
- **Annual saving: ~$1,860 per 50 clients**

The saving scales linearly with client volume.

### Verifying cache hits

Every API response includes debug headers:

```
X-Cache-Read-Tokens: 1847    ← tokens served from cache (you paid ~10% of normal)
X-Cache-Created-Tokens: 0    ← tokens written to cache this call
X-PII-Redacted: false        ← whether PII was detected in this message
```

These headers are visible in browser DevTools → Network → Response Headers. This is the token optimisation audit trail — Bitcoin Suisse can verify cache performance in real time.

### Response body also includes debug info

```json
{
  "response": "...",
  "agent": "relationship_manager",
  "piiRedacted": false,
  "_debug": {
    "requestId": "550e8400...",
    "tokensIn": 143,
    "tokensOut": 312,
    "cacheRead": 1847,
    "cacheCreated": 0
  }
}
```

---

## 3. Combined compliance posture

When both mechanisms are active:

1. Client sends message via their advisory interface
2. Message enters the regulated institution's server
3. Anonymisation middleware strips PII → audit log entry created
4. Anonymised message + agent mandate sent to Claude API
5. STATIC_CONTEXT served from Anthropic's 5-minute cache (no re-transmission of the large block)
6. Response returned to server
7. Server logs token usage, cache hit/miss, PII flag
8. Response displayed to client via the advisory interface

**What Anthropic receives:** anonymised portfolio context + the static BPF/thesis block (research data, not client data).

**What the regulated institution retains:** the full audit log, the original message (pre-anonymisation), the response, and the token usage record.

**What the regulator can inspect:** the full server-side audit trail, the anonymisation configuration, and the agent system prompts (open in this repository).
