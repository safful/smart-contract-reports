## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- sponsor confirmed
- selected for report
- M-15

# [Oracle assumes token and feed decimals will be limited to 18 decimals](https://github.com/code-423n4/2022-10-inverse-findings/issues/533) 

# Lines of code

https://github.com/code-423n4/2022-10-inverse/blob/main/src/Oracle.sol#L87
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Oracle.sol#L121


# Vulnerability details

## Impact

The `Oracle` contract normalizes prices in both `viewPrices` and `getPrices` functions to adjust for potential decimal differences between feed and token decimals and the expected return value. 

However these functions assume that `feedDecimals` and `tokenDecimals` won't exceed 18 since the normalization calculation is `36 - feedDecimals - tokenDecimals`, or that at worst case the sum of both won't exceed 36.

This assumption should be safe for certain cases, for example WETH is 18 decimals and the ETH/USD chainlink is 8 decimals, but may cause an overflow (and a revert) for the general case, rendering the Oracle useless in these cases.

## Proof of Concept

If `feedDecimals + tokenDecimals > 36` then the expression `36 - feedDecimals - tokenDecimals` will be negative and (due to Solidity 0.8 default checked math) will cause a revert.

## Recommended Mitigation Steps

In case `feedDecimals + tokenDecimals` exceeds 36, then the proper normalization procedure would be to **divide** the price by `10 ** decimals`. Something like this:

```
uint normalizedPrice;

if (feedDecimals + tokenDecimals > 36) {
    uint decimals = feedDecimals + tokenDecimals - 36;
    normalizedPrice = price / (10 ** decimals)
} else {
    uint8 decimals = 36 - feedDecimals - tokenDecimals;
    normalizedPrice = price * (10 ** decimals);
}
```