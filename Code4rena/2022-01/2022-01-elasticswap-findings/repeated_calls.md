## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Repeated calls](https://github.com/code-423n4/2022-01-elasticswap-findings/issues/178) 

# Handle

pauliax


# Vulnerability details

## Impact
Result of this.totalSupply() could be cached to avoid duplicate calls:
```solidity
  require(this.totalSupply() > 0, "Exchange: INSUFFICIENT_LIQUIDITY");
  ...
  uint256 totalSupplyOfLiquidityTokens = this.totalSupply();
```


