## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Division by zero in `_computeLiquidityFromAdjustedBalances` of `HybridPool`](https://github.com/code-423n4/2021-09-sushitrident-findings/issues/185) 

# Handle

broccoli


# Vulnerability details

## Impact

The `_computeLiquidityFromAdjustedBalances` function of `HybridPool` should return in the `if (s == 0)` statement, or it will cause a divison-by-zero error otherwise.

## Proof of Concept

Referenced code:
[HybridPool.sol#L350-L352](https://github.com/sushiswap/trident/blob/9130b10efaf9c653d74dc7a65bde788ec4b354b5/contracts/pool/HybridPool.sol#L350-L352)

## Recommended Mitigation Steps

Add `return computed;` after `computed = 0;`.

