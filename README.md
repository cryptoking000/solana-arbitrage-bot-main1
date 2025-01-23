# Rust Macros and Arbitrage Bots on Solana

## Introduction

When starting with Rust, macros might seem like a magical yet elusive feature. However, they can shine in practical use cases, especially while building advanced applications like arbitrage bots on Solana. In this guide, we'll explore how Rust macros simplify the development of on-chain programs for Solana-based arbitrage bots through a practical example.

---

## What is Arbitrage?

Arbitrage involves trading tokens across different exchanges (DEXs/AMMs) with varying prices to make a profit. Here's a simple example:

- **Market A:** BTC price = \$40,000
- **Market B:** BTC price = \$50,000

You could buy 1 BTC on Market A for \$40,000 and sell it on Market B for \$50,000, resulting in a \$10,000 profit:

- **Steps:** \$40K USD → 1 BTC → \$50K USD
- **Profit:** \$50K - \$40K = \$10K

---

## Atomic Transactions on Solana

In Solana, an arbitrage bot includes multiple swap instructions (ixs) in a transaction (tx). A key instruction checks profitability, ensuring that the transaction reverts if there's no profit. For example:

```text
[
  buy(market_A, 1BTC),
  sell(market_B, 1BTC),
  require(new_usd_balance > init_usd_balance)
]
```

The `require` instruction ensures that the transaction and all preceding instructions are reverted if the condition isn't satisfied.

---

## Accounting for Slippage

Slippage occurs when the actual trade outcome differs from the quoted amount due to market changes. For instance:

- Quoted: 1 BTC
- Actual: 0.99 BTC

To address this, Solana requires tracking swap states across trades. Here's an example sequence:

```text
[
  set swap_state = 40K,
  buy(market_A, swap_state),
  sell(market_B, swap_state),
  require(init_usd_balance < new_usd_balance)
]
```

The `swap_state` updates dynamically based on trade outcomes. This state management logic must run on-chain within a custom program.

---

## Developing an On-Chain Program

A simplified on-chain program function might look like this:

```rust
fn swap_market_A(accounts) {
    // Compute: amount_in = swap_state
    // Perform: swap on market A with amount_in
    // Save: swap_state = amount_post_swap - amount_pre_swap
}
```

Each exchange requires a unique function due to differing account structures. For example:

```rust
pub fn orca_swap<'info>(ctx: Context<'_, '_, '_, 'info, OrcaSwap<'info>>) -> Result<()> {
    // Prepare, swap, and record output
    _orca_swap(ctx, amount_in);
}

pub fn saber_swap<'info>(ctx: Context<'_, '_, '_, 'info, SaberSwap<'info>>) -> Result<()> {
    // Prepare, swap, and record output
    _saber_swap(ctx, amount_in);
}
```

---

## Using Rust Macros to Simplify Code

Writing separate functions for each exchange leads to duplication. Ideally, a generic function could handle this:

```rust
pub fn amm_swap(swap_fcn: F<**???**>, ctx: Context<**???**>) {
    let amount_in = prepare_swap(&ctx.accounts.swap_state);
    let amount_pre_swap = ctx.accounts.user_dst.balance;

    swap_fcn(&ctx, amount_in);

    let swap_state = &mut ctx.accounts.swap_state;
    let amount_post_swap = ctx.accounts.user_dst.balance;
    end_swap(swap_state, amount_post_swap - amount_pre_swap);
}
```

Rust's strict typing makes defining a generic `Context` challenging. Enter macros, which can dynamically generate the required code.

---

## Solution: Rust Macros

Here's a macro to write a swap function based on the `Context` type:

```rust
#[macro_export]
macro_rules! basic_amm_swap {
    ($swap_fcn:expr, $typ:ident < $tipe:tt > ) => {{
        |ctx: Context<'_, '_, '_, 'info, $typ<$tipe>> | -> Result<()> {
            let amount_in = prepare_swap(&ctx.accounts.swap_state).unwrap();
            $swap_fcn(&ctx, amount_in).unwrap();

            let swap_state = &mut ctx.accounts.swap_state;
            let user_dst = &mut ctx.accounts.user_dst;
            end_swap(swap_state, user_dst).unwrap();

            Ok(())
        }
    }};
}
```

### Usage Example

Define specific swap functions using the macro:

```rust
pub fn orca_swap<'info>(ctx: Context<'_, '_, '_, 'info, OrcaSwap<'info>>) -> Result<()> {
    basic_amm_swap!(_orca_swap, OrcaSwap<'info>)(ctx)
}

pub fn saber_swap<'info>(ctx: Context<'_, '_, '_, 'info, SaberSwap<'info>>) -> Result<()> {
    basic_amm_swap!(_saber_swap, SaberSwap<'info>)(ctx)
}
```

This approach reduces boilerplate code and ensures consistency across functions.

---

## Conclusion

Rust macros are a powerful tool for code generation, particularly in scenarios requiring repetitive and type-specific logic, like Solana arbitrage bots. By leveraging macros, we create scalable, maintainable, and efficient code, unlocking the full potential of on-chain trading strategies.

