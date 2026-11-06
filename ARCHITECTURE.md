# Velana Mining — Architecture & Flow (devnet)

How the Velana mining system fits together: the on-chain program, the
off-chain coordinator, and this web UI. Written so a new collaborator can
understand the whole loop and connect to the on-chain program.

## The big picture

```
Miner (browser / Phantom)
   │ 1. stake 1,000 tokens (Holder tier, no NFT needed)
   ▼
On-chain program (Solana devnet) ── holds stake + reward vault
   │ 2. miner asks coordinator for a puzzle
   ▼
Coordinator (off-chain server) ── serves puzzles, verifies solutions
   │ 3. miner solves the 3-SAT puzzle locally (DPLL)
   ▼
Coordinator verifies the solution, then signs + sends `submit` on-chain
   │ 4. program credits 2.2 tokens (2 base + 10% staking bonus) to pending
   ▼
Miner: claim → withdraw (linear vesting to month end)
```

The program never trusts the miner directly for rewards — only the
coordinator's signature can attest a solve. The coordinator cannot steal
funds: it can only attest valid solves, and it can refuse to attest (censor),
but it cannot forge a reward or move anyone's tokens.

## On-chain (Solana devnet)

| Item | Value |
|---|---|
| Mining program | `AirHpAYsRh1DKDWXL6oRy9kgrhBnLbqta8jgNnhBKMre` |
| Game program | `Ey6tiMVAgTnGKwTBrSabMRKyLJKChWKrjm6eau1HEyzE` |
| Velana token mint | `6gWvj9qsLGXvCvaSEBFkHnRA71iRfkeonDzysb7uwJXT` (6 decimals) |
| RPC | `https://api.devnet.solana.com` |
| Metaplex Core (NFT passes) | `CoREENxT6tW1HoK8ypY1SxRMGpY9tYx6nBBhKxg4nFq` |
| Coordinator pubkey | `FrQAmbvpdByGC4wqZqKZnJegJjK1h3MBDsDdRm9v5k5A` |
| Admin / deploy wallet | `6xEmHhC1KTedK3i7v7oEcdjnJcGouKCwHCFohYdqYsjh` |

### Program accounts (PDAs)

| PDA | Seeds | Purpose |
|---|---|---|
| Config | `["config"]` | All parameters: quotas, fees, coordinator, mint, collections |
| Vault | `["vault"]` | Holds the 500M mining reward tokens (no withdraw function) |
| Stake vault | `["stake_vault"]` | Holds staked tokens |
| Miner | `["miner", wallet]` | Per-miner state: tier, staked amount, lock_until, solves, pending, nonce |
| Lock | `["lock", wallet]` | Per-miner vesting lock for claimed rewards |
| Lock vault | `["lock_vault", wallet]` | Token account for the vesting lock |

### Instructions

| Instruction | Who calls it | What it does |
|---|---|---|
| `initialize` | admin, once | Creates Config + Vault + Stake vault |
| `stake(tier, amount, nft_asset?)` | miner | Locks tokens (+ NFT pass for Regular/Special) in the program. NFT is transferred into program custody (escrow) and returned on unstake |
| `submit(puzzle_id, nonce)` | **coordinator only** | Credits the reward to the miner's pending balance. Requires `nonce == miner.next_nonce`, then increments it (replay guard). Requires an active (unexpired) stake and remaining daily quota |
| `claim` | miner | Moves pending rewards into the miner's vesting lock (linear unlock to month end). Pending older than the claim window is burned instead |
| `withdraw` | miner | Withdraws the vested (unlocked) portion from the lock. A 5% tax applies if withdrawn < 5 days after the previous withdraw (burned) |
| `unstake` | anyone (permissionless crank) | Returns the stake after the lock expires. NFT return is best-effort; if it fails, `recover_nft` retries it |
| `recover_nft` | miner | Retries returning a stuck NFT after unstake |
| `close_month` | anyone (permissionless crank) | Burns the unused monthly emission (`cap - used`) and resets the counter. Only callable after the month rolls over. The caller gains nothing — tokens are burned, not paid out |
| `update_config` | admin | Changes cap, fees, coordinator, min stake, withdraw tax. Does **not** change quotas, stake lock, or month length (those are set at `initialize`) |

### Tiers and rewards

| Tier | Value | Quota (design) | Requirement |
|---|---|---|---|
| Regular | 1 | 4 solves/day | NFT pass (10K @ $1 + admission puzzle) + 1,000 tokens staked |
| Special | 2 | 8 solves/day | NFT pass (5K @ $2, no puzzle) + 1,000 tokens staked |
| Holder | 3 | 1 solve/day | No NFT — stake 1,000 tokens only |

- Reward: **2 tokens per solve**, flat. Stakers get **+10%** (2.2 total).
- Staking an extra 1,000 tokens as an NFT-pass holder adds **+1 solve/day**.
- Submit fee: **$0.01 in SOL** per submit, charged as debt (max $2 = 200 submits).
- Staking model: tokens are locked for **1 calendar month** (fixed 30-day
  periods anchored at the unix epoch). No early unstake. Return is via the
  permissionless `unstake` crank.
- One Agent ID = one miner, one tier only.

### Current devnet config (TEST values — not production)

The deployed devnet config uses short values so testing is fast:

| Parameter | Devnet (live) | Production (design) |
|---|---|---|
| `month_secs` | 120 s (2 min) | 30 days |
| `stake_lock_secs` | 120 s (2 min) | 1 month (30 days) |
| `quotaHolder` | 10 / day | 1 / day |
| `claim_window_secs` | 600 s (10 min) | 30 days |
| `early_withdraw_window_secs` | 120 s (2 min) | 5 days |
| `monthly_cap` | 4,440,000 | 4,440,000 |
| `min_stake_tokens` | 1,000 | 1,000 |
| `reward_base` / `bonus` | 2 / +10% | 2 / +10% |

Because the test month is only 2 minutes, `close_month` can be called often
and burns ~4.44M tokens per call — that is the test config, not a bug. The
program enforces whatever the config says. Fixing these values requires a
program change (`update_config` cannot touch them) followed by an upgrade.

### How to connect to the on-chain program

1. **Wallet:** Phantom (or any Solana wallet). Switch it to **Devnet**
   (Settings → Developer Settings → Change Network → Devnet). Mainnet will
   not work — the program does not exist there.
2. **RPC:** `https://api.devnet.solana.com` (public devnet). Use
   `finalized` commitment for reads right after a transaction, because
   devnet reads can briefly return pre-transaction state.
3. **SDK:** Anchor 0.32.1 + `@solana/web3.js` 1.98. The program IDL is
   `velana_mining.json` (in the coordinator repo / program source).
4. **Read the config first:** fetch the Config PDA, then derive the Miner's
   PDA with seeds `["miner", wallet]` to see tier, stake, `next_nonce`,
   pending rewards, and quota.
5. **Tokens:** devnet Velana is a test token (mint above). Airdrop it from
   the deploy wallet or the faucet flow used for testing.
6. **Never** submit a solve directly — `submit` requires the coordinator's
   signature. Send your solution to the coordinator instead (below).

## Off-chain: the coordinator

The coordinator is a small Node.js/Express server. It is the only signer
allowed to call `submit`.

Live: **https://velana-coordinator.fly.dev/**
Source (with the keypair): `D:/Project/velana-solana/velana-coordinator/`
(and `~/workspace/velana-coordinator/` on the dev machine).

### What it does

1. **Serves puzzles** — a pool of planted 3-SAT instances (100 vars,
   426 clauses, ratio ~4.26, balanced planting so no polarity bias leaks).
2. **Verifies solutions** — checks the submitted assignment satisfies every
   clause (~1 ms). Invalid solutions are rejected with `INVALID_SOLUTION`.
3. **Attests on-chain** — reads the miner's `next_nonce` from the program,
   checks the daily quota, then signs and sends `submit(puzzle_id, nonce)`
   with the coordinator keypair. The reward lands in the miner's pending.

### API

| Endpoint | Method | Description |
|---|---|---|
| `/health` | GET | Status, coordinator pubkey, program ID, puzzle count |
| `/puzzles` | GET | List of available puzzle IDs |
| `/puzzle/:id` | GET | Returns `{ puzzle_id, vars, clauses, cnf }` (DIMACS CNF text) |
| `/submit` | POST | Body `{ miner_wallet, puzzle_id, assignment }`. `assignment` is an array of ints, one per variable: positive = true, negative = false (DIMACS style). Success returns `{ success, signature, solscan, puzzle_id, nonce, pending_rewards, solves_today, next_nonce }` |

### Running it yourself

```bash
cd velana-coordinator
npm install
node server.js   # listens on :3000 (or $PORT)
```

- Keypair: `coordinator-keypair.json` in the same folder, or the
  `COORDINATOR_KEYPAIR` env var (JSON array). **Devnet test key only —
  keep it safe, never share it. Mainnet needs proper custody (vault/HSM).**
- The deployed Fly.io instance reads the keypair from a Fly secret
  (`COORDINATOR_KEYPAIR`), not from the image.

### Security model (read this before changing the coordinator)

- The coordinator **cannot steal funds**. It can only sign `submit`, which
  credits rewards to the miner who solved. It cannot move tokens or submit
  for a wallet that did not solve.
- Its failure modes are: refusing to attest (censorship) and being
  compromised to attest fake solves (which would mint rewards without work).
  That is why the keypair must be guarded and rotated for mainnet.
- The program's replay guard is the per-miner `next_nonce`: a submit with a
  stale nonce reverts with `InvalidNonce`.

## This web UI

Live: **https://velana-mining.fly.dev/** — single self-contained `index.html`
(Anchor.js + web3.js bundled, Buffer polyfill included).

- Connects Phantom (devnet), reads live Config / Vault / Miner / Lock.
- Actions: Stake, Claim, Withdraw, Unstake, Recover NFT, Close Month.
- **Submit Solve** is wired to the coordinator: it fetches a puzzle from
  `velana-coordinator.fly.dev`, solves it in-browser with a DPLL solver
  (~0.5 s for a 100-var puzzle), then POSTs the assignment to the
  coordinator, which verifies and submits on-chain. The tx signature and
  Solscan link appear in the log.
- The page must be opened directly or self-hosted (Fly.io/Netlify/Vercel).
  A Muse share-link iframe blocks the page's fetches to the Solana RPC.

## Full mining loop (end-to-end, verified on devnet)

1. Miner stakes 1,000 tokens (Holder, tier 3). Miner PDA created.
2. Miner fetches a puzzle: `GET /puzzle/2` from the coordinator.
3. Miner solves it (Kissat / the web's DPLL) → assignment.
4. Miner posts: `POST /submit { miner_wallet, puzzle_id: 2, assignment }`.
5. Coordinator verifies (~1 ms), reads `next_nonce`, submits on-chain.
   Pending reward: **2.2 tokens** (2 + 10%). `solvesToday` +1, `nextNonce` +1,
   submit-fee debt 0.0001 SOL.
6. Miner calls `claim` → 2.2 tokens move into the vesting lock.
7. Miner calls `withdraw` → vested portion released (linear to month end).

Verified 2026-11-06 with a burner wallet (stake → solve → submit → claim →
2 withdrawals). The program works end-to-end. The remaining known gaps are
the test config values above and the open decisions (coordinator key custody,
program upgrade-authority policy).

## Repo layout

```
index.html          # the mining web UI (single file)
README.md           # short project readme
ARCHITECTURE.md     # this file — offchain + onchain flow
```
