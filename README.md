# amm-anchor: a constant-product AMM on Solana, built with Anchor

A minimal Uniswap-v2-style automated market maker for two SPL tokens, written in Rust with Anchor. It's part of my **Solana from first principles** course: the code here is what we build on camera, line by line.

> **Watch the lessons:** [Part 1](https://youtu.be/FwcrPbUakj8) · [Part 2](https://youtu.be/8Bt0a7KWz0o) · [Part 3](https://youtu.be/cqGn0EempdE)

By the end you'll understand:

- how a pool holds two tokens in program-owned vaults (PDAs)
- the `x · y = k` swap math, fees and slippage protection
- how LP tokens represent a share of the pool, and how deposits and withdrawals are priced
- how to test it all end to end against a local validator (Surfpool)

## What the program does

| Instruction | Who calls it | What it does |
| --- | --- | --- |
| `initialize(seed, fee, authority)` | pool creator | Creates the pool config, the LP mint and the two token vaults |
| `deposit(amount, max_x, max_y)` | liquidity provider | Adds X and Y to the pool and mints `amount` LP tokens |
| `swap(is_x, amount, min)` | trader | Swaps X→Y (or Y→X) along the curve, failing if output is below `min` |
| `withdraw(amount, min_x, min_y)` | liquidity provider | Burns `amount` LP tokens and returns a pro-rata share of X and Y |
| `lock()` / `unlock()` | pool authority | Pauses or resumes deposits, swaps and withdrawals |

Every instruction emits an event (`InitializeEvent`, `SwapEvent`, …) so indexers and frontends can follow pool activity.

## Accounts

```
Config PDA        seeds = ["config", seed (u64 LE)]
 ├─ LP mint PDA   seeds = ["lp", config]        mint authority = Config
 ├─ vault_x       ATA(mint_x, owner = Config)
 └─ vault_y       ATA(mint_y, owner = Config)
```

- **Config** stores the two mints, the fee (in basis points), an optional `authority` and a `locked` flag. The `seed` lets one program host many pools.
- **Vaults** are associated token accounts owned by the Config PDA, so only the program can move tokens out, by signing with the Config seeds.
- **LP mint** is itself a PDA whose mint authority is the Config, so LP tokens can only be minted by `deposit`.

`has_one = mint_x, mint_y` constraints on every instruction make sure the mints passed in match the pool.

## The math

The pool keeps the product of its reserves constant:

```
x · y = k
```

**Swap.** Swapping `Δx` of X in (after a fee `f`, in basis points) gives out:

```
Δy = y − k / (x + Δx · (1 − f / 10_000))
```

The trader sets `min`: if `Δy < min` the transaction fails, which protects against price movement between signing and landing (slippage).

**Deposit.** The first deposit sets the price: the depositor chooses the X and Y amounts and the LP amount. After that, to mint `L` LP tokens against a supply `S`, a depositor must add the same share of each reserve:

```
x_in = X · L / S        y_in = Y · L / S
```

`max_x` / `max_y` cap what they're willing to pay.

**Withdraw.** Burning `L` LP tokens returns the same share:

```
x_out = X · L / S       y_out = Y · L / S
```

`min_x` / `min_y` set the least they'll accept.

The curve math comes from [`constant-product-curve`](https://github.com/deanmlittle/constant-product-curve).

## Run it

Prerequisites: Rust, the Solana CLI, Anchor, Node.js + Yarn, and [Surfpool](https://surfpool.run).

```bash
git clone https://github.com/princedotrs/amm-anchor.git
cd amm-anchor
yarn install
anchor build

# terminal 1: local validator
surfpool start

# terminal 2: run the tests
anchor test --skip-local-validator
```

The test suite (`tests/amm-anchor.ts`) walks through the full lifecycle: create two mints, initialize a pool with a 0.3% fee, deposit, lock and unlock, swap both directions, and withdraw. Event listeners print each emitted event.

## Project layout

```
programs/amm-anchor/src/
├── lib.rs              # instruction entry points + events
├── state/mod.rs        # Config account
├── instructions/
│   ├── initialize.rs
│   ├── deposit.rs
│   ├── swap.rs
│   ├── withdraw.rs
│   └── update.rs       # lock / unlock
├── events.rs
├── error.rs            # AmmError (+ mapping from CurveError)
└── constants.rs
tests/amm-anchor.ts     # end-to-end tests
```

## Ideas to extend it

- Derive the Config from both mints (`["config", mint_x, mint_y, seed]`) so each pair has one canonical pool
- Support Token-2022 via `TokenInterface` and `transfer_checked`
- On the first deposit, mint `sqrt(x · y)` LP tokens instead of a caller-chosen amount
- Let the authority update the fee or hand over authority

## License

MIT
