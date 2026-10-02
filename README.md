# Venus — Solana Smart Trading Bot

A Python Telegram bot that lets someone buy and sell Solana tokens in one tap
— and, the reason it exists, hand a position to **Smart Trade**: the bot buys,
takes profits in stages, protects the position with a stop-loss that only ever
moves up, and reports every step in plain language.

Built, deployed and operated solo: trading engine, infrastructure, website and
launch material. It ran publicly on Telegram for six weeks with real money on
the line, and was then deliberately shut down — see
[Why it was paused](#why-it-was-paused).

> This repository is the public write-up. **The implementation is private**:
> it generates and holds users' private keys, and publishing that code would
> be irresponsible regardless of the project's status.

---

## At a glance

| | |
|---|---|
| **Live** | 4 Aug – 13 Sep 2026, Telegram, real funds |
| **Stack** | Python, aiogram 3, Jupiter swap API, Solana RPC, systemd on a Hetzner VPS |
| **Scale** | ~12.8k lines, single process, no database |
| **Tests** | 16 standalone suites, including source-level loop-parity and strict-compile gates |
| **Trades** | 87 executed, all owner testing · 5 users created wallets |
| **Shut down because** | custodial trading is a regulated activity a one-person project cannot meet |

---

## The interesting part: one stop-loss that only moves up

Three strategy presets plus a fully custom one (five numbers: stop-loss, three
take-profit targets, trailing distance):

| Strategy | Stop-loss | Target 1 | Target 2 | Target 3 |
|---|---|---|---|---|
| 🛡️ Conservative | −8%, trails 10% below peak | +12%, sell 40% | +25%, sell 30% | +40%, sell 20% |
| ⚖️ Standard | −12%, trails 18% | +20%, sell 25% | +45%, sell 25% | +80%, sell 20% |
| 🚀 Degen | −25%, trails 30% | +50%, sell 20% | +120%, sell 20% | +250%, sell 20% |

Protection starts below the buy price, trails the highest price seen at the
strategy's distance, and ratchets upward every time a target is hit — after
the first take-profit it sits above entry, so the position can no longer lose
money. It never moves down. After the final target the remainder rides the
trend under the same trail.

Each 30-second tick: quote the position → glitch guard → resume
reconciliation → new-high/trail → stop-loss → targets. **State is persisted
before any notification is sent**, so a crash mid-sale can never replay a sell
or lower a stop.

## Three bugs worth the whole test suite

Money-handling code fails in ways that are invisible until they are expensive.
These are the ones that shaped how the project was built:

- **The +86,792% take-profit.** A single corrupted quote from the swap API
  briefly reported an absurd price, and the engine dutifully sold into it.
  Fix: a glitch guard that holds any quote more than 10× or less than 0.1× the
  last accepted one until a *second* quote confirms it. Trading logic now
  treats its own price feed as untrusted input.
- **A stray `(` that compiled fine.** An f-string typo produced
  `'str' object is not callable` — syntactically valid, and it would have
  crashed at the first third-target hit, mid-sale. Fix: the pre-deploy suite
  compiles the bot with `SyntaxWarning` promoted to an error.
- **Two monitor loops that drifted.** Fresh positions and positions resumed
  after a restart ran through separate loops that had to behave identically.
  They stopped being identical. Fix: a test that diffs the two loops' source
  and fails on divergence.

## Judgment calls

- **A feature built, reviewed, and deleted.** A "burn dust" tool for clearing
  worthless token balances passed basic testing, then an adversarial review
  surfaced 19 confirmed findings — NFTs, LP tokens, and, during a swap-API
  outage, *the entire portfolio* could be classified as dust and burned. It
  was removed rather than patched, and the write-up says not to rebuild it.
- **Never sell blind.** A stop-loss or take-profit is never executed on an
  assumed price. During an RPC outage the bot states that it cannot see the
  market rather than acting on stale data.
- **Telegram can never break trading.** Notifications are strictly
  best-effort; a messaging failure cannot abort or alter a trade.
- **Receipts are permanent.** Every action leaves a screen that is never
  overwritten, with the amounts, the resulting protection level, and a
  block-explorer link.

## Architecture

- **One process, one file, no database.** Per-user state lives in small JSON
  files written atomically (`tmp` + `os.replace`) with rotating backups for the
  wallet store — deliberately boring persistence for something that must
  survive a hard restart mid-trade.
- **Custodial wallets**, up to three per user, private keys encrypted at rest
  with a key supplied from the environment.
- **Swaps via Jupiter**: quote → build transaction → sign locally → submit via
  RPC, rate-paced at 8/s, with unroutable mints cached and a
  token→SOL→USD price fallback.
- **Telegram concurrency**: same-user updates are serialised, because without
  it a slow tap's state reset could wipe an input flow mid-typing. Handler
  registration order is enforced by a test.

## Designed for people who do not use crypto

Plain language, one concept per screen, and the parts that usually bite
newcomers handled explicitly: tokens that arrived without a Venus buy are
flagged as likely airdrop scams, empty token accounts can be cleaned up to
recover rent, an underfunded buy shows a friendly "not enough SOL" screen with
the funding address instead of an RPC error, and slippage is capped.

Alongside: live positions with average-cost profit and loss, balances with USD
estimates, trending and top-gainer discovery with a liquidity floor,
per-user fee and timezone settings, trade history, and referrals that pay out
as fee discounts rather than cash.

## Money model

0.9% per swap — collected inside the swap on sells, as a native transfer on
buys, because in-swap fees fail on many Token-2022 routes. Receipts show
amounts net of fees. Cost basis is average-cost per user per token, reduced by
every sell. Lifetime revenue was **≈0.027 SOL across 87 trades**, essentially
all of it the owner's own testing, against roughly $75/month of infrastructure
— an honest reflection of a product that was shut down before it found users,
not a business.

## Operations history

- **2026-05** password-gated prototype, 3 testers
- **2026-07** full feature build; real-money testing begins
- **2026-08-04** deployed to a Hetzner VPS under a hardened systemd unit, with
  a daily backup cron for the wallet store
- **2026-08-18** marketing site live on a custom domain
- **2026-08 → 09** iteration driven by live use: one-screen take-profit
  receipts, the price-glitch guard, a cost-basis fix, the unified trailing
  stop-loss — each money-touching change through an adversarial review round
  before deploy
- **2026-09-13** paused: service stopped, bot token revoked, site unpublished

## Why it was paused

Venus is **custodial**. It generated and held users' private keys and executed
trades on their behalf, for a fee, for anyone on Telegram, worldwide. In most
jurisdictions that profile falls under money-transmission, crypto-asset-service
or investment-service rules — MiCA in the EU among others — carrying
registration, KYC/AML and consumer-protection obligations that a one-person
project cannot genuinely meet. Nothing in the codebase addressed them.

Shutting down a working product is a worse outcome than never building it only
if you ignore what it was becoming. The engineering problem — making automated
trading safe enough to hand someone else's money to — was the interesting
part, and it is solved in here. The regulatory problem is not an engineering
problem, and pretending otherwise is how side projects turn into liabilities.
