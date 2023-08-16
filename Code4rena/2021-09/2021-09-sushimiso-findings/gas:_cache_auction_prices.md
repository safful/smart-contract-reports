## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: Cache auction prices](https://github.com/code-423n4/2021-09-sushimiso-findings/issues/105) 

# Handle

cmichel


# Vulnerability details

The `DutchAuction.clearingPrice` function can save gas by caching the computed prices instead of recomputing it.

## Recommended Mitigation Steps
Cache the values:

```solidity
function clearingPrice() public view returns (uint256) {
    /// @dev If auction successful, return tokenPrice
    uint256 _tokenPrice = tokenPrice();
    uint256 _currentPrice = priceFunction();
    return _tokenPrice > _currentPrice ? _tokenPrice : _currentPrice;
}
```


