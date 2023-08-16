## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [`ConcentratedLiquidityPool`s can be created with the same tokens](https://github.com/code-423n4/2021-09-sushitrident-2-findings/issues/33) 

# Handle

cmichel


# Vulnerability details

The `ConcentratedLiquidityPool.constructor` does not check that `_token0 != _token1`.
The pool factory does not ensure this either.

## Impact
Pools can be created using the same token.
This should be prevented as it does not make sense.

## Recommended Mitigation Steps
Add a `_token0 != _token1` check to the constructor.


