## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: `HybridPool._computeLiquidityFromAdjustedBalances` should return early](https://github.com/code-423n4/2021-09-sushitrident-findings/issues/103) 

# Handle

cmichel


# Vulnerability details

The `HybridPool._computeLiquidityFromAdjustedBalances` function should return early if `s == 0` as it will always return zero.
Currently, it still performs an expensive loop iteration.

```solidity
if (s == 0) {
  // gas: should do an early return here
  computed = 0;
  // return 0;
}
```

## Recommended Mitigation Steps
Return early with a value of `0` if `s == 0`.


