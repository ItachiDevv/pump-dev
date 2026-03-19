---
name: pump-dev
description: >
  Expert knowledge for pump.fun development — fee systems, on-chain programs, creator fee claims,
  GitHub social fee PDAs, PumpSwap AMM, token lifecycle, and API integration.
  Use when building on pump.fun, monitoring on-chain events, integrating with pump.fun APIs,
  or working with the @pump-fun/pump-sdk.
metadata:
  author: nirholas
  version: "1.0"
  tags: pump.fun, solana, defi, fees, claims, pumpswap, github
---

# pump.fun Development Expert

Comprehensive knowledge of pump.fun's on-chain programs, fee systems, APIs, and SDKs.

## Three-Program Architecture

| Program | ID | Purpose |
|---------|-----|---------|
| **Pump** (bonding curve) | `6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P` | Token creation, buy/sell on bonding curve, creator fee collection, fee distribution |
| **PumpSwap AMM** | `pAMMBay6oceH9fJKBRHGP5D4bD4sWpmSwMn52FMfXEA` | Post-graduation AMM pools, coin creator fees, cashback |
| **PumpFees** | `pfeeUxB6jkeY1Hxd7CsFCAjcbHA9rWtchMGdZ6VojVZ` | Fee sharing config, social fee PDAs (GitHub/X), dynamic fee tiers |

### Known Accounts

| Account | Address | Role |
|---------|---------|------|
| Fee recipient | `CebN5WGQ4jvEPvsVU4EoHEpgzq1VV7AbCJ5GEFDM97zC` | Protocol fee account |
| Migration authority | `39azUYFWPz3VHgKCf3VChUwbpURdCHRxjWVowf5jUJjg` | Token graduation authority |
| Social claim authority | `2sMrGNK8i36YRkF5WWCwnaUYuwDJhHe1g2xA8aPvhkjM` | Signs all social fee claims |
| Crank wallet | `9xgqzhT1pzLvjR5VwxP1yZ2z5NpNAtXcdGV7GPHvHR4Z` | Auto-distribution sweeper |

## Token Lifecycle

1. **Creation** — Token launched on bonding curve (Pump program)
2. **Bonding curve trading** — Buy/sell with fees routed to creator vault
3. **Graduation** — Token migrates to PumpSwap AMM pool when bonding curve completes
4. **AMM trading** — Constant-product swaps, fees split: LP + protocol + creator

## Fee Structure (Dynamic, Market-Cap Based)

Fees are tiered by market cap via the PumpFees program `FeeConfig`:

| Market Cap (SOL) | Total Fee | Creator Fee | LP Fee | Protocol Fee |
|---|---|---|---|---|
| 0 - 420 | 1.25% | 1.00% | 0.20% | 0.05% |
| ... (scales down) | ... | ... | ... | ... |
| 98,240+ | 0.30% | 0.05% | 0.20% | 0.05% |

### Fee Types at Token Launch (Permanent)

- **Creator Fee coin** — creator earns a % of trading fees, claimable anytime
- **Cashback coin** — 100% of creator fees redirected to traders as cashback
- **This choice is permanent and cannot be changed after launch**

## Fee Claim Flow (Three Phases)

### Phase 1: Accumulation (automatic, per-trade)
Every trade routes fees to the `creator_vault` PDA (native SOL on bonding curve) or `coin_creator_vault_ata` (WSOL on PumpSwap). No explicit instruction needed.

### Phase 2: Distribution
`distributeCreatorFees` reads the `SharingConfig` for a token and splits fees among up to 10 shareholders. **Permissionless** — anyone can call it.

### Phase 3: Social Fee Claim
`claim_social_fee_pda` transfers SOL from a social fee PDA to the user's wallet. **Requires `social_claim_authority` co-signature** (pump.fun's server key).

## Instruction Discriminators

| Instruction | Discriminator (hex) | Program |
|---|---|---|
| `collect_creator_fee` | `1416567bc61cdb84` | Pump |
| `claim_cashback` | `253a237ebe35e4c5` | Pump / PumpSwap |
| `distribute_creator_fees` | `a572670079cef751` | Pump |
| `collect_coin_creator_fee` | `a039592ab58b2b42` | PumpSwap |
| `transfer_creator_fees_to_pump` | `8b348655e4e56cf1` | PumpSwap |
| `claim_social_fee_pda` | `e115fb85a11ec7e2` | PumpFees |

## Event Discriminators

| Event | Discriminator (hex) |
|---|---|
| `CollectCreatorFeeEvent` | `7a027f010ebf0caf` |
| `DistributeCreatorFeesEvent` | `a537817004b3ca28` |
| `ClaimCashbackEvent` | `e2d6f62107f293e5` |
| `CollectCoinCreatorFeeEvent` | `e8f5c2eeeada3a59` |
| `SocialFeePdaClaimed` | `3212c141edd2eaec` |
| `CreateFeeSharingConfigEvent` | `8569aac8b874fb58` |
| `UpdateFeeSharesEvent` | `15bac4b85be4e1cb` |

## SocialFeePdaClaimed Event Layout

```
disc(8) + timestamp(i64=8) + user_id(borsh string: 4-byte LE len + N UTF-8 bytes) +
platform(u8) + social_fee_pda(32) + recipient(32) + social_claim_authority(32) +
amount_claimed(u64) + claimable_before(u64) + lifetime_claimed(u64) +
recipient_balance_before(u64) + recipient_balance_after(u64)
```

- `platform`: 0=pump, 1=twitter, **2=GitHub**
- `user_id`: GitHub numeric user ID as string (e.g., "146933685")
- First claim detection: `lifetime_claimed == amount_claimed`
- Fake claim detection: `amount_claimed == 0`

## DistributeCreatorFeesEvent Layout

```
disc(8) + timestamp(i64=8) + mint(32) + sharingConfig(32) + admin(32) +
shareholders(vec: 4-byte count + N * {pubkey(32) + share_bps(u16)}) + distributedAmount(u64)
```

**The mint is at offset 16** — this is how you resolve which token a claim belongs to.

## SharingConfig Account Layout

```
disc(8) [216,74,9,0,56,140,93,75] + bump(1) + version(1) + status(1) +
mint(32, offset 11) + admin(32, offset 43) + admin_revoked(1) +
shareholders(vec at offset 76: count(4) + [address(32) + share_bps(u16)]...)
```

## PDA Seeds

| PDA | Seeds | Program |
|-----|-------|---------|
| `feeProgramGlobal` | `["fee-program-global"]` | PumpFees |
| `sharingConfig` | `["sharing-config", mint]` | PumpFees |
| `socialFeePda` | Derived from user_id + platform | PumpFees |
| `bondingCurve` | `["bonding-curve", mint]` | Pump |
| `global` | `["global"]` | Pump |
| `globalConfig` | `["global_config"]` | PumpSwap |
| `creatorVaultAuthority` | `["creator_vault", coinCreator]` | PumpSwap |
| `feeConfig` | `["fee_config", configProgramId]` | PumpFees |

## GitHub Social Fee Claim Patterns (CRITICAL)

Three on-chain transaction patterns for `claim_social_fee_pda`, distinguished by **fee payer**:

### Pattern A — Crank Auto-Sweep (SKIP)
- Fee payer = crank wallet (`9xgqzhT1...`)
- Recipient = crank wallet
- Auto-distribution to pump.fun's system, not a user claim
- **Always skip these**

### Pattern B — AUTH Combined (ACCEPT)
- Fee payer = social_claim_authority (`2sMrGNK8...`)
- AUTH is sole signer
- **Always** has `DistributeCreatorFees` in same tx (100% of observed cases)
- Mint extractable from the distribution event
- Used for first claims and some subsequent claims via pump.fun web UI

### Pattern C — User Split (ACCEPT)
- Fee payer = user's own wallet
- 2 signers: user wallet + AUTH co-signs
- **Never** has `DistributeCreatorFees` in same tx
- Distribution happened in a separate preceding tx (3-4 seconds earlier)
- Mint resolved from PDA transaction history (most recent `DistributeCreatorFees` on that PDA)
- User's wallet = fee payer = recipient in event

### Detection Rule
```typescript
const feePayer = tx.transaction.message.accountKeys[0]; // first writable signer
if (feePayer === CRANK_WALLET) return; // Pattern A — skip
// Pattern B or C — accept, resolve mint accordingly
```

### Mint Resolution
- **Pattern B**: Extract from `DistributeCreatorFees` event in same tx (mint at offset 16)
- **Pattern C**: Look at PDA's recent tx history for the preceding `DistributeCreatorFees`

## pump.fun API

### Base URL
`https://frontend-api-v3.pump.fun`

### Token Info
```
GET /coins/{mint}
```
Returns: `name`, `symbol`, `market_cap` (SOL), `usd_market_cap` (USD), `is_currently_live`, etc.

**IMPORTANT**: `market_cap` is in SOL, `usd_market_cap` is in USD. Always use `usd_market_cap` for display.

### Other Endpoints
```
GET /coins/{mint}/top-holders    — Top 20 holders
GET /coins/{mint}/trades/latest  — Recent trades
GET /users/{address}             — Creator profile
GET /coins?searchTerm={name}&sort=market_cap — Search tokens
```

### Fallback: DexScreener API
When pump.fun API returns empty (graduated tokens), use:
```
GET https://api.dexscreener.com/latest/dex/tokens/{mint}
```
Returns `pairs[0].baseToken.name`, `pairs[0].baseToken.symbol`, `pairs[0].marketCap` (USD).

## SDK

### Official SDK
```bash
npm install @pump-fun/pump-sdk
```

Key classes:
- `PumpAmmSdk` — PumpSwap AMM interactions
- `collectCoinCreatorFee(coinCreator)` — Claim AMM creator fees
- `getCoinCreatorVaultBalance(coinCreator)` — Check claimable balance

### Social Fee PDA Helpers
- `createSocialFeePda(user_id, platform)` — Initialize a social fee PDA
- `createSharingConfigWithSocialRecipients(...)` — Set up fee sharing with GitHub users
- Event decoding for all on-chain events from Pump, PumpAMM, and PumpFees programs

## Monitoring On-Chain Events

### WebSocket `onLogs`
```typescript
import { Connection, PublicKey } from '@solana/web3.js';

const PUMP_FEES = new PublicKey('pfeeUxB6jkeY1Hxd7CsFCAjcbHA9rWtchMGdZ6VojVZ');
const conn = new Connection(rpcUrl, { wsEndpoint: wsUrl });

conn.onLogs(PUMP_FEES, (logs) => {
  if (logs.err) return;
  if (logs.logs.some(l => l.includes('ClaimSocialFeePda'))) {
    // Process claim
  }
}, 'confirmed');
```

### HTTP Polling Fallback
```typescript
const sigs = await conn.getSignaturesForAddress(PUMP_FEES, { limit: 20, until: lastSig });
```

**WARNING**: PumpFees program gets ~500 txs/second (CPI calls from every swap). Filter aggressively.

### Helius Webhooks (Recommended)
Monitor the `social_claim_authority` account — only fires on actual claim transactions:
```bash
helius webhook create \
  --url "https://your-app.com/helius-webhook" \
  --accounts "2sMrGNK8i36YRkF5WWCwnaUYuwDJhHe1g2xA8aPvhkjM" \
  --types "ANY" --webhook-type "enhanced"
```

## GitHub User Resolution

The `SocialFeePdaClaimed` event gives a numeric GitHub user ID. Resolve to username:
```
GET https://api.github.com/user/{id}
```

X/Twitter handle: parse from `twitter_username` field in GitHub profile.

## Common Gotchas

1. **pump.fun API returns empty for some tokens** — Use DexScreener as fallback
2. **`market_cap` vs `usd_market_cap`** — Former is SOL, latter is USD. Always use USD.
3. **Social fee claims are custodial** — Require pump.fun's `social_claim_authority` to co-sign. Users cannot claim outside pump.fun's app.
4. **GitHub orgs are NOT supported** as social fee recipients — only individual users who can log in
5. **A single social fee PDA can receive fees from multiple tokens** — GPA scans return multiple SharingConfigs. Use the `DistributeCreatorFees` event for accurate mint resolution.
6. **`DistributeCreatorFees` is permissionless** — Anyone can trigger it. When any shareholder claims, all shareholders get distributed to.
7. **Cashback coins cannot have CTOs** — The cashback choice is permanent at launch.

## Reference Documentation

- [Official Fee Docs](https://pump.fun/docs/fees)
- [pump-public-docs](https://github.com/pump-fun/pump-public-docs)
- [PumpSwap Creator Fee README](https://github.com/pump-fun/pump-public-docs/blob/main/docs/PUMP_SWAP_CREATOR_FEE_README.md)
- [SDK (npm)](https://www.npmjs.com/package/@pump-fun/pump-sdk)
- [DeepWiki Analysis](https://deepwiki.com/pump-fun/pump-public-docs)
