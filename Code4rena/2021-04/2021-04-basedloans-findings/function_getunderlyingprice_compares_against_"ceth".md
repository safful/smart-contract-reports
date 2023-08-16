## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [function getUnderlyingPrice compares against "cETH"](https://github.com/code-423n4/2021-04-basedloans-findings/issues/26) 

# Handle

paulius.eth


# Vulnerability details

## Impact
contract CompoundLens functions cTokenMetadata and cTokenBalances compare against "bETH" while contract SimplePriceOracle function getUnderlyingPrice compares against "cETH". It is not clear if this SimplePriceOracle will be used in production, probably only for testing, but still would be nice to unify it across all the contracts.

## Recommended Mitigation Steps
Replace "cETH" with "bETH" in SimplePriceOracle function getUnderlyingPrice.

