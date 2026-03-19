# PumpSwap AMM — Deep Reference

## Overview

PumpSwap is pump.fun's constant-product (x*y=k) AMM. Tokens "graduate" here after completing their bonding curve.

Program ID: `pAMMBay6oceH9fJKBRHGP5D4bD4sWpmSwMn52FMfXEA`

## Pool Structure

```typescript
interface Pool {
  poolBump: number;
  index: number;
  creator: PublicKey;
  baseMint: PublicKey;    // token mint
  quoteMint: PublicKey;   // WSOL
  lpMint: PublicKey;
  poolBaseTokenAccount: PublicKey;
  poolQuoteTokenAccount: PublicKey;
  lpSupply: BN;
  coinCreator: PublicKey;
  isMayhemMode: boolean;
  isCashbackCoin: boolean;
}
```

## Creator Fee Collection

Fees do NOT go directly to the creator during swaps. They accumulate in a **creator vault PDA**.

### Creator Vault PDA
- Seeds: `["creator_vault", coinCreator]` under PumpSwap program
- Holds an ATA for WSOL (quote token)

### collect_coin_creator_fee
- Discriminator: `[160, 57, 89, 42, 181, 139, 43, 66]` (hex: `a039592ab58b2b42`)
- **No arguments** — claims entire vault balance
- **Permissionless** — anyone can call (but fees always go to the coin creator)
- Accounts: quote_mint, quote_token_program, coin_creator, coin_creator_vault_authority, coin_creator_vault_ata, coin_creator_token_account

### transfer_creator_fees_to_pump
- Moves WSOL fees from PumpSwap vault to Pump program's creator vault
- Used before `distributeCreatorFees` for graduated tokens

## Key Instructions (25 total)

Buy/sell, deposit/withdraw (LP), create_pool, collect_coin_creator_fee, claim_cashback, transfer_creator_fees_to_pump, set_coin_creator, migrate_pool_coin_creator, etc.

## Events

- `BuyEvent` / `SellEvent` — include `coinCreator`, `coinCreatorFeeBasisPoints`, `coinCreatorFee`
- `CollectCoinCreatorFeeEvent` — `{ timestamp, coinCreator, coinCreatorFee, vaultAta, tokenAccount }`
- `ClaimCashbackEvent` — `{ user, amount, timestamp, totalClaimed, totalCashbackEarned }`

## GlobalConfig

```typescript
interface GlobalConfig {
  admin: PublicKey;
  lpFeeBasisPoints: number;
  protocolFeeBasisPoints: number;
  coinCreatorFeeBasisPoints: number;
  protocolFeeRecipients: PublicKey[];  // 8 round-robin recipients
  // ...
}
```

## Fee Calculation

For pump pools (created by bonding curve authority), fees are **tiered by market cap** using `FeeConfig.feeTiers[]`. For non-pump pools, flat fees from `FeeConfig.flatFees`.

```typescript
fee = ceilDiv(amount * basisPoints, 10000)
// If coinCreator == PublicKey.default (all zeros), creator fee = 0
```
