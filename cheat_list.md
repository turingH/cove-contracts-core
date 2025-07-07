# Auditor Prior Knowledge Cheat List

This file summarizes recurring patterns and repository-specific details that can help rule out false positives during
automated reviews.

## Deposit Processing in Cove Contracts

- **\_processPendingDeposits** (`src/libraries/BasketManagerUtils.sol` lines 1137‑1179) computes the shares to mint and
  updates `basketBalanceOf` without touching external strategies.
- When `basketValue` and `totalSupply` are zero, the minted shares equal
  `pendingDepositValue` returned by `eulerRouter.getQuote` (`src/libraries/BasketManagerUtils.sol`
  lines 1163‑1168). `pendingDepositValue` uses 18‑decimal USD precision so the
  share supply aligns with the 18‑decimal `decimals()` in `BasketToken.sol`
  (lines 1125‑1130).
- Deposits are finalized by calling `BasketToken.fulfillDeposit`, which transfers the deposited tokens from the
  `BasketToken` contract to the BasketManager (`src/BasketToken.sol` lines 460‑494).
- `proposeTokenSwap` invokes `_processPendingDeposits` at the start of a rebalance
  (`src/libraries/BasketManagerUtils.sol` lines 268‑295) so the basket balance already includes any pending deposits.
- **No function withdraws assets from a strategy to fulfill deposits.** Idle balances are handled entirely within the
  BasketToken and BasketManager contracts.
- Reports that the protocol ignores idle funds or pulls liquidity from strategies during deposits are false unless these
  lines change.

## Redemption Processing

- Pending redemptions are processed in `BasketManagerUtils.sol` lines 648‑678. The function converts the
  USD value owed to redeemers back into the basket's base asset using
  `eulerRouter.getQuote` (lines 659‑663) and stores the result in `withdrawAmount`.
- `withdrawAmount` is capped by the BasketManager's base‑asset balance (lines 664‑665) before
  calling `BasketToken.fulfillRedeem` to burn shares and transfer assets.
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

## AssetRegistry Responsibilities

- `AssetRegistry` (`src/AssetRegistry.sol`) only manages asset lists and their enabled or paused status.
- It exposes helpers like `addAsset`, `setAssetStatus`, `getAssets` and `getAssetsBitFlag` but **no price conversion logic**.
- Conversions between token amounts and USD occur exclusively in `BasketManagerUtils` via `EulerRouter.getQuote`
  (see `_processPendingDeposits` lines 1147‑1179 and `_calculateBasketValue` lines 1188‑1207).
- Reports mentioning `_convertToUSD` or `_convertFromUSD` inside `AssetRegistry` are false unless new code adds such functions.

## External Trade Weight Enforcement

- `completeRebalance` (`src/libraries/BasketManagerUtils.sol` line 395) ends the rebalance after the mandatory delay.
- `_processExternalTrades` (line 420) applies the amounts returned from `_completeTokenSwap` to `basketBalanceOf`.
- The nested loops inside this function increment their counters (`++j` at line 761 and `++i` at line 766) so execution always progresses; reports of missing increments causing an infinite loop are false unless these lines change.
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
  force infinite rebalance loops. Reports of contradictory thresholds causing DoS are outdated unless `_isRebalanceRequired` returns.

## Rebalance Weight Snapshot and Retry Limit

- `proposeRebalance` caches each basket's `getTargetWeights()` when forming
  `basketTargetWeights` (`src/libraries/BasketManagerUtils.sol` lines 255‑261) and
  hashes them into `rebalanceStatus.basketHash` (line 329).
- `completeRebalance` verifies these arrays with `_validateBasketHash`
  (line 409) and never re-fetches weights from the strategies during the
  process.
- When `_isTargetWeightMet` fails, the rebalance status resets to
  `REBALANCE_PROPOSED` and `retryCount` is incremented (lines 431‑438).
- After `retryLimit` attempts (`src/BasketManager.sol` line 178 sets the default
  limit to 3), `_finalizeRebalance` runs even if weights still deviate.
- Dynamic weight strategies therefore cannot cause indefinite DoS; at worst,
  completion is delayed by `retryLimit * stepDelay` seconds.

## BasketToken BitFlag Management

- Basket tokens are deployed as clones of a fixed implementation stored in
  `_bmStorage.basketTokenImplementation` when `BasketManager` is constructed
  (`src/BasketManager.sol` lines 146‑176). This implementation address cannot be
  changed later.
- Each `BasketToken` stores its asset selection in the `bitFlag` state variable
  (`src/BasketToken.sol` line 124) which is set during `initialize` when the
  basket is created (`src/libraries/BasketManagerUtils.sol` lines 170‑223).
- The only way to modify a basket's bitFlag after creation is via
  `BasketManager.updateBitFlag` (`src/BasketManager.sol` lines 606‑659), which is
  restricted to the `_TIMELOCK_ROLE` and simultaneously updates the manager's
  `basketAssets` mappings.
- `BasketToken.setBitFlag` (`src/BasketToken.sol` lines 496‑504) enforces that
  only the Basket Manager can change the flag by calling `_onlyBasketManager`
  (`src/BasketToken.sol` lines 669‑672).
- Because the bitFlag cannot be arbitrarily altered or provided by user-controlled
  implementations, checks such as
  `AssetRegistry.hasPausedAssets(BasketToken(basket).bitFlag())` in
  `proposeRebalance` (`src/libraries/BasketManagerUtils.sol` line 265) correctly
  reflect the assets held by the basket. Reports that the pause mechanism can be
  bypassed by tampering with `BasketToken.bitFlag()` are therefore false unless
  these lines change.

## Basket Array Length Verification

- `_validateBasketHash` (`src/libraries/BasketManagerUtils.sol` lines 994-1021) is called from `proposeTokenSwap` (line 362) and `completeRebalance` (line 409).
- The helper reverts with `BasketsMismatch` unless `baskets.length`, `basketsTargetWeights.length`, and `basketAssets.length` are all equal and every `basketAssets[i].length` matches `basketsTargetWeights[i].length`.
- `_isTargetWeightMet` relies on these checks before looping over `basketAssets[i][j]` using the target weight length (lines 1045‑1128). Thus any mismatch fails fast with `BasketsMismatch`, preventing out-of-bounds reads.
- Reports that a malformed weight array can freeze rebalances via array overflows are false unless these lines change.
