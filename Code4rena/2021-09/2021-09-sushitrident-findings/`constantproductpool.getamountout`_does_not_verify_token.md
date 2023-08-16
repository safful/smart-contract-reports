## Tags

- bug
- sponsor confirmed
- 0 (Non-critical)

# [`ConstantProductPool.getAmountOut` does not verify token](https://github.com/code-423n4/2021-09-sushitrident-findings/issues/93) 

# Handle

cmichel


# Vulnerability details

The `ConstantProductPool.getAmountOut` function does not verify that `tokenIn == token1` in the `else` branch.
This is done everywhere else though (see `swap` and `flashSwap`) and should be done here as well.

## Impact
The function can be called with a token that is not any of the pool tokens.

## Recommended Mitigation Steps
Add the missing check.

