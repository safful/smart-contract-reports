## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [`DateTime.getMarketIndex` bounds should be tighter](https://github.com/code-423n4/2021-08-notional-findings/issues/82) 

# Handle

cmichel


# Vulnerability details

## Vulnerability Details
`DateTime.getMarketIndex` can be called with a `maxMarketIndex < 10` but the inner `DateTime.getTradedMarket(i)` function will revert for any values `i > 7`.

## Impact
"Valid" `maxMarketIndex` values above 7 will break and return with an error.

## Recommended Mitigation Steps
The upper bound on `maxMarketIndex` should be set to `7`.

