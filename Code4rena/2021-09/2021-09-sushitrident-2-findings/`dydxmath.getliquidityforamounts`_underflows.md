## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [`DyDxMath.getLiquidityForAmounts` underflows](https://github.com/code-423n4/2021-09-sushitrident-2-findings/issues/30) 

# Handle

cmichel


# Vulnerability details

The `DyDxMath.getLiquidityForAmounts/getDx/getDy` functions perform unchecked computations on `priceUpper - priceLower` but they do not check that `priceUpper >= priceLower`.

## Impact
The values can underflow and return much lower liquidity or much higher token amounts than expected.

The calling functions (`mint` and `burn`) also do not check this. For `mint`, it fails further down the callstack at `Ticks.insert`, but `burn` does not fail.

## Recommended Mitigation Steps
Check that the `lower` and `upper` from the provided parameters for `mint` and `burn` are indeed sorted, i.e., `lower < upper`.
It should be checked explicitly at the start of the function.


