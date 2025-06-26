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

## Oracle Quote Precision

- All price oracles in the repository operate using **18‑decimal** precision (WAD).
- `AnchoredOracle` defines `_WAD = 1e18` (`src/oracles/AnchoredOracle.sol` lines 20‑23) and passes values in this format when fetching and validating quotes.
- The testing `MockPriceOracle` returns quotes scaled by 1e18 (`test/utils/mocks/MockPriceOracle.sol` lines 13‑23), matching the production expectation.
- `BasketManagerUtils` converts asset balances to USD by calling `EulerRouter.getQuote` and then mixes those results with `_WEIGHT_PRECISION = 1e18` (`src/libraries/BasketManagerUtils.sol` lines 60‑64 and 1188‑1207).
- Therefore, any report claiming that oracle quotes use a 6‑decimal format or that a 6/18 decimal mismatch exists is incorrect unless new code explicitly changes the quote precision.
