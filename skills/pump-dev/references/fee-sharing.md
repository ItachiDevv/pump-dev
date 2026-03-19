# Fee Sharing System — Deep Reference

## SharingConfig

Up to 10 shareholders can split creator fees. Each gets a share in basis points (1/10,000), totaling exactly 10,000 BPS (100%).

### Creating a Sharing Config
```typescript
import { PumpSdk } from '@pump-fun/pump-sdk';

// With wallet shareholders
await sdk.createFeeSharingConfig(mint, admin, shareholders);

// With social (GitHub/X) recipients
await sdk.createSharingConfigWithSocialRecipients(mint, admin, socialRecipients);
```

### Updating Shares
```typescript
await sdk.updateFeeShares(mint, admin, newShareholders);
// NOTE: updateFeeShares automatically distributes pending fees first
```

### Revoking Admin
```typescript
await sdk.revokeFeeSharingAuthority(mint, admin);
// After revocation, shares can never be changed again (CTO lock)
```

## Social Fee PDA

A `SocialFeePda` is created for each social identity (GitHub user, X user) that receives fees:

```
Seeds: derived from user_id + platform
Account: { bump, version, user_id (string max 20), platform (u8), total_claimed (u64), last_claimed (u64), _reserved[128] }
```

Platform enum: `0 = pump, 1 = twitter, 2 = GitHub`

### Claim Rate Limiting
The `FeeProgramGlobal` account has a `claim_rate_limit` field, suggesting rate limiting on claim frequency. The `social_claim_authority` is stored here and can only be changed by the program authority.

## Fee Distribution Flow

When `distributeCreatorFees` is called:
1. Reads the `SharingConfig` for the token mint
2. Calculates each shareholder's portion based on their BPS
3. Transfers SOL to each shareholder address
4. For social fee PDA shareholders, SOL goes to the PDA (not directly to a wallet)
5. Emits `DistributeCreatorFeesEvent` with the mint, shareholders, and distributed amount

The instruction is **permissionless** — anyone can call it. This is why "triggered rewards distribution" messages appear automatically in the pump.fun app timeline.

## CTO (Community Takeover)

When a token creator abandons a project, the community can perform a CTO:
1. `updateFeeShares` — redirect 100% of fees to a new community wallet
2. `revokeFeeSharingAuthority` — permanently lock the shares

**Cashback coins cannot have CTOs** — the fee structure is permanently locked at launch.
