# Auditor Prior Knowledge Cheat List

This file summarizes recurring patterns and repository-specific details that can help rule out false positives during
automated reviews.

## Deposit Processing in Cove Contracts

- **\_processPendingDeposits** (`src/libraries/BasketManagerUtils.sol` lines 1137‑1179) computes the shares to mint and
  updates `basketBalanceOf` without touching external strategies.
- Deposits are finalized by calling `BasketToken.fulfillDeposit`, which transfers the deposited tokens from the
  `BasketToken` contract to the BasketManager (`src/BasketToken.sol` lines 460‑494).
- `proposeTokenSwap` invokes `_processPendingDeposits` at the start of a rebalance
  (`src/libraries/BasketManagerUtils.sol` lines 268‑295) so the basket balance already includes any pending deposits.
- **No function withdraws assets from a strategy to fulfill deposits.** Idle balances are handled entirely within the
  BasketToken and BasketManager contracts.
- Reports that the protocol ignores idle funds or pulls liquidity from strategies during deposits are false unless these
  lines change.

## Redemption Processing

- Pending redemptions are processed in `BasketManagerUtils.sol` lines 648‑678. The function calculates a
  `withdrawAmount` capped by the BasketManager's base-asset balance (lines 664‑665) and then calls
  `BasketToken.fulfillRedeem` to burn shares and transfer assets.
- No function pulls funds from external strategies when fulfilling redemptions; the assets are already held by the
  BasketManager.

Use these references when assessing reports about deposit or redemption logic.

## Oracle Quote Precision

- All price oracles operate using **18‑decimal WAD precision**.
- `AnchoredOracle` sets `_WAD = 1e18` (`src/oracles/AnchoredOracle.sol` lines 20‑23) and validates quotes using this
  scale.
- `MockPriceOracle` in the test suite multiplies prices by `1e18` before returning a value
  (`test/utils/mocks/MockPriceOracle.sol` lines 13‑23).
- `BasketManagerUtils` relies on `_WEIGHT_PRECISION = 1e18` (`src/libraries/BasketManagerUtils.sol` lines 60‑64) and
  converts balances to USD via `EulerRouter.getQuote` in `_calculateBasketValue` (lines 1188‑1207), which expects
  18‑decimal inputs.
- Thus, claims of a 6‑decimal oracle format or mismatched precision are false unless the code around these lines is
  modified.

## External Trade Weight Enforcement

- `completeRebalance` (`src/libraries/BasketManagerUtils.sol` line 395) ends the rebalance after the mandatory delay.
- `_processExternalTrades` (line 420) applies the amounts returned from `_completeTokenSwap` to `basketBalanceOf`.
- The balances are then re-evaluated via `_initializeBasketData` and `_isTargetWeightMet` (lines 428‑444). If any basket weight deviates beyond `weightDeviationLimit`, the function reverts and sets the status back to `REBALANCE_PROPOSED`.
- `_validateExternalTrades` (lines 908‑991) merely simulates trades using their `minAmount`; the final weight check in `completeRebalance` must still pass with real claimed amounts.
- As a result, finalization without calling `executeTokenSwap` is only possible when the baskets already meet their target weights or when the retry limit (`self.retryLimit`) has been reached.  Otherwise `_isTargetWeightMet` fails and the rebalance restarts.

## Token Swap Execution Authorization

- `BasketManager.executeTokenSwap` (`src/BasketManager.sol` lines 412‑447) is protected by `onlyRole(_TOKENSWAP_EXECUTOR_ROLE)`. Unauthorized callers will revert via AccessControl.
- `proposeTokenSwap` (`src/libraries/BasketManagerUtils.sol` lines 347‑377) sets `rebalanceStatus.status` to `TOKEN_SWAP_PROPOSED` and stores a hash of the proposed trades.
- `executeTokenSwap` verifies that the status is `TOKEN_SWAP_PROPOSED` and the hash matches before setting `rebalanceStatus.status` to `TOKEN_SWAP_EXECUTED` and calling the adapter.
- `completeRebalance` only processes external trades when the status is `TOKEN_SWAP_EXECUTED`; otherwise the final weight check described above enforces execution.
- Any report suggesting this function is publicly callable or lacks authorization checks is incorrect unless these lines change.

## Swap Adapter Path Processing

- Token swaps are implemented solely through `CoWSwapAdapter`. This contract loops over each `ExternalTrade` once and increments the loop counter after every iteration.
- `executeTokenSwap` uses `for (uint256 i = 0; i < externalTrades.length;)` and increments `i` inside the loop (`src/swap_adapters/CoWSwapAdapter.sol` lines 79‑95).
- `completeTokenSwap` follows the same pattern (`src/swap_adapters/CoWSwapAdapter.sol` lines 106‑137).
- There is no function named `_processSwapPath` or any branch that `continue`s without advancing the index. Claims of an infinite loop due to skipped tokens are false unless new code introducing such logic appears.
- The balances are then re-evaluated via `_initializeBasketData` and `_isTargetWeightMet` (lines 428‑444). If any basket
  weight deviates beyond `weightDeviationLimit`, the function reverts and sets the status back to `REBALANCE_PROPOSED`.
- `_validateExternalTrades` (lines 908‑991) merely simulates trades using their `minAmount`; the final weight check in
  `completeRebalance` must still pass with real claimed amounts.

## Token Swap Execution Authorization

- `BasketManager.executeTokenSwap` (`src/BasketManager.sol` lines 412‑447) is protected by
  `onlyRole(_TOKENSWAP_EXECUTOR_ROLE)`. Unauthorized callers will revert via AccessControl.
- The function validates the current status is `TOKEN_SWAP_PROPOSED` and ensures the passed `externalTrades` hash
  matches the stored proposal before setting the status to `TOKEN_SWAP_EXECUTED` and delegate-calling the adapter.
- Any report suggesting this function is publicly callable or lacks authorization checks is incorrect unless these lines
  change.

## Rebalance Threshold Logic

- Commit `0b02ae8` removed the `_isRebalanceRequired` helper that used raw balance differences to trigger rebalances.
- `proposeTokenSwap` now relies solely on `_isTargetWeightMet` and reverts when weights deviate beyond
  `weightDeviationLimit` (see lines 378‑385 of `src/libraries/BasketManagerUtils.sol`).
- `completeRebalance` also checks `_isTargetWeightMet` before finalising the rebalance (lines 430‑444 of the same file).
- Since rebalances are entered and exited using the same weight deviation check, dust‑level deposits or redeems cannot
  force infinite rebalance loops. Reports of contradictory thresholds causing DoS are outdated unless
  `_isRebalanceRequired` returns.