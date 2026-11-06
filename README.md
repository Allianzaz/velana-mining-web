# Velana Mining Web

Single-file web UI for the Velana mining program on **Solana devnet**.

- Live site: https://velana-mining.fly.dev/
- Open `index.html` directly in a browser (no build step).

## What it does

- Connects Phantom (devnet) and reads live program state: Config, Vault, Miner, Lock.
- Actions: Stake, Claim, Withdraw, Unstake, Recover NFT, Close Month.
- **Submit Solve is wired to the coordinator** — fetches a puzzle from `velana-coordinator.fly.dev`, solves it in-browser with a DPLL solver, and the coordinator verifies + submits on-chain. See `ARCHITECTURE.md` for the full offchain + onchain flow.
- Local 3-SAT puzzle practice (generator + DPLL solver).

## Program (devnet)

- Mining program: `AirHpAYsRh1DKDWXL6oRy9kgrhBnLbqta8jgNnhBKMre`
- Velana token mint: `6gWvj9qsLGXvCvaSEBFkHnRA71iRfkeonDzysb7uwJXT` (6 decimals)
- Tiers: Regular = 1, Special = 2, Holder = 3 (Holder stakes tokens only, no NFT pass).
- Min. stake: 1,000 tokens. Reward: 2 tokens per solve.

## Notes

- The page must be opened directly or self-hosted (Fly.io/Netlify/Vercel). A Muse share-link iframe blocks the page's fetches to the Solana RPC.
- Devnet only — test tokens, no real value.
