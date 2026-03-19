# Claim Monitoring — Implementation Reference

## Architecture for Monitoring GitHub Fee Claims

### Detection Stack (Triple-Layered)

1. **Helius Webhook** (fastest) — Monitor `social_claim_authority` account. Helius pushes matching txs to your endpoint. Only fires on claim-related txs.
2. **WebSocket `onLogs`** (real-time) — Subscribe to PumpFees program. Filter for `"Program log: Instruction: ClaimSocialFeePda"` in log lines.
3. **HTTP Polling** (fallback) — `getSignaturesForAddress` on PumpFees program every 30s. Catches anything WebSocket missed.

### Processing Pipeline

```
Detect tx signature
  → Dedup check (Redis SET with 24h TTL)
  → Fetch full tx: getParsedTransaction(sig)
  → Extract SocialFeePdaClaimed event from "Program data:" log lines
  → Filter: platform == 2 (GitHub), amount > 0 (not fake)
  → Filter: fee payer != crank wallet (not auto-sweep)
  → Resolve token mint:
      Pattern B: from DistributeCreatorFees in same tx
      Pattern C: from PDA transaction history
  → Enrich: pump.fun API + GitHub API + X API
  → Persist to database
  → Fan out notifications
```

### Transaction Pattern Detection

```typescript
const accountKeys = tx.transaction.message.accountKeys;
const firstKey = accountKeys[0];
const feePayer = /* extract pubkey string from firstKey */;

const CRANK = '9xgqzhT1pzLvjR5VwxP1yZ2z5NpNAtXcdGV7GPHvHR4Z';
const AUTH = '2sMrGNK8i36YRkF5WWCwnaUYuwDJhHe1g2xA8aPvhkjM';

if (feePayer === CRANK) {
  // Pattern A: auto-sweep → SKIP
  return;
}

// Pattern B (AUTH) or Pattern C (user) → ACCEPT
let mint = extractMintFromDistributeEvent(tx.meta.logMessages);

if (!mint) {
  // Pattern C: look at PDA's recent tx history
  mint = await resolveMintFromPdaHistory(connection, socialFeePda);
}
```

### Mint Resolution from PDA History

For Pattern C claims (no DistributeCreatorFees in same tx):

```typescript
async function resolveMintFromPdaHistory(conn, pdaAddress) {
  const sigs = await conn.getSignaturesForAddress(new PublicKey(pdaAddress), { limit: 20 });
  const DIST_DISC = Buffer.from('a537817004b3ca28', 'hex');
  const mintTotals = new Map();

  for (const sig of sigs) {
    const tx = await conn.getParsedTransaction(sig.signature, { ... });
    for (const log of tx.meta.logMessages) {
      if (!log.startsWith('Program data: ')) continue;
      const buf = Buffer.from(log.slice('Program data: '.length), 'base64');
      if (buf.length >= 48 && buf.subarray(0, 8).equals(DIST_DISC)) {
        const mint = new PublicKey(buf.subarray(16, 48)).toBase58();
        const amount = Number(buf.readBigUInt64LE(buf.length - 8));
        mintTotals.set(mint, (mintTotals.get(mint) || 0) + amount);
      }
    }
  }

  // Return mint with highest total distributed (primary token)
  let best = null, bestAmt = 0;
  for (const [mint, total] of mintTotals) {
    if (total > bestAmt) { bestAmt = total; best = mint; }
  }
  return best;
}
```

### WebSocket Resilience

- Heartbeat every 60s
- Reconnect if silent for 90s
- RPC fallback rotation across multiple endpoints
- Exponential backoff on 429 errors

### Notification Delivery (BullMQ)

Pre-format message once, fan out 1 job per active chat:
- Rate limit: 25 msg/sec (leave headroom below Telegram's 30/sec)
- Custom backoff: 5s → 30s → 2min → 2min → 2min
- DLQ sweep every 5 min, exhausted after 8 total attempts
- Auto-deactivate chats after 2 consecutive 403s

## Enrichment Sources

| Data | Source | Endpoint |
|------|--------|----------|
| Token name, symbol, mcap | pump.fun API | `GET /coins/{mint}` |
| Token fallback | DexScreener | `GET /latest/dex/tokens/{mint}` |
| GitHub username | GitHub REST | `GET /user/{id}` |
| GitHub stars | GitHub REST | `GET /users/{name}/repos?sort=stars` |
| X handle | GitHub profile | `twitter_username` field |
| X followers | RapidAPI twitter-api47 | `GET /v3/user/by-username?username={handle}` |

## Volume Considerations

The PumpFees program processes ~500 txs/second (CPI calls from every swap for fee calculations). Real `claim_social_fee_pda` events are rare — maybe dozens per day. Filter aggressively:

- WebSocket: check log lines for `"Instruction: ClaimSocialFeePda"` before fetching full tx
- Polling: `getSignaturesForAddress` returns ALL program txs — most are not claims
- Helius webhook on `social_claim_authority`: only fires on claim txs (most efficient)
