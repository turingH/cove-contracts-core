# Auditor Prior Knowledge Cheat List

This file summarizes recurring patterns and repository-specific details that can help rule out false positives during automated reviews.

## Deposit Processing in Cove Contracts

- **_processPendingDeposits** (`src/libraries/BasketManagerUtils.sol` lines 1137‑1169) only calculates how many shares to mint and updates internal balances. It mints shares via `BasketToken.fulfillDeposit` and **does not withdraw liquidity from any external strategy or vault**.
- When `proposeTokenSwap` handles a rebalance (lines 268‑295 of the same file), it calls `_processPendingDeposits` with the current basket state. The function simply adds the pending deposit amount to the basket’s base asset balance in storage and mints shares.
- **No code path pulls funds from a strategy to satisfy these deposits.** Tokens used to fulfill deposits are already held in the `BasketToken` contract and are transferred to the BasketManager inside `BasketToken.fulfillDeposit` (`src/BasketToken.sol` lines 460‑494).
- Therefore, issues claiming that idle balances are ignored when withdrawing to fulfill pending deposits are unfounded unless new code introduces such a withdrawal.

## Redemption Processing

- Pending redemptions are fulfilled by calculating a withdrawal amount (lines 648‑678 of `BasketManagerUtils.sol`) and calling `BasketToken.fulfillRedeem`, which burns shares and transfers assets from the BasketManager back to the token. This withdrawal logic is separate from the deposit flow above.

Use these references when assessing reports about deposit or redemption logic.
