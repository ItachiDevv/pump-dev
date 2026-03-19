# pump-dev

An [Agent Skill](https://agentskills.io) that teaches AI agents how to build on [pump.fun](https://pump.fun) — the Solana token launchpad.

## What's Included

Expert knowledge of pump.fun's:
- **Three-program architecture** — Pump (bonding curve), PumpSwap (AMM), PumpFees (fee sharing)
- **Fee system** — Dynamic fee tiers, creator fees, cashback coins, fee sharing configs
- **GitHub social fee claims** — Social fee PDAs, claim patterns, on-chain detection
- **APIs** — pump.fun frontend API, DexScreener fallback, GitHub/X enrichment
- **SDK** — `@pump-fun/pump-sdk` integration patterns
- **Monitoring** — WebSocket, polling, Helius webhook patterns for real-time event detection

## Install

```bash
# Add to any compatible AI agent
npx skills add nirholas/pump-dev
```

## Skill Structure

```
skills/pump-dev/
├── SKILL.md                        # Main skill (programs, fees, events, patterns)
└── references/
    ├── fee-sharing.md              # SharingConfig, social PDAs, CTOs
    ├── claim-monitoring.md         # Real-time claim detection architecture
    └── pumpswap-amm.md            # AMM pools, creator vaults, fee collection
```

## Key Topics

- On-chain program IDs, instruction discriminators, event discriminators
- SocialFeePdaClaimed event layout (Borsh decoding)
- Three claim transaction patterns (Pattern A/B/C) and filtering logic
- Token mint resolution from DistributeCreatorFees events
- pump.fun API (`market_cap` vs `usd_market_cap` gotcha)
- Production monitoring with Helius webhooks

## License

MIT
