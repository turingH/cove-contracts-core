- **Rebalance Length Mismatch**: `_isTargetWeightMet` assumed `basketAssets[i]` and `basketsTargetWeights[i]` had equal lengths. A crafted proposal with extra weight entries triggers an out-of-bounds read during finalisation, reverting and locking the protocol in the `TOKEN_SWAP_PROPOSED` state. Root cause: lack of per-basket length checks on rebalance input arrays.

