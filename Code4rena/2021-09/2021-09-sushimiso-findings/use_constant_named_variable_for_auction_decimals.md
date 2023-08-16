## Tags

- bug
- 0 (Non-critical)
- disagree with severity
- sponsor confirmed
- resolved

# [Use constant named variable for auction decimals](https://github.com/code-423n4/2021-09-sushimiso-findings/issues/103) 

# Handle

cmichel


# Vulnerability details

The `CrowdSale.initCrowdsale` function checks that the auction token has 18 decimals through `IERC20(_token).decimals() == 18`.
This seems to be related to `AUCTION_TOKEN_DECIMALS` and these values should not get ouf of sync.

## Impact
These values can easily get out of sync.

## Recommended Mitigation Steps
Create another named constant and set it to `18` decimals:

```solidity
uint256 private constant AUCTION_TOKEN_DECIMAL_PLACES = 18;
uint256 private constant AUCTION_TOKEN_DECIMALS = 10 ** AUCTION_TOKEN_DECIMAL_PLACES;
```


