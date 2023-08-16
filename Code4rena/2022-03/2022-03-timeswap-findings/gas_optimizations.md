## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Gas Optimizations](https://github.com/code-423n4/2022-03-timeswap-findings/issues/37) 

2022-03-timeswap gas optimization

1 Use memory for cache instead of storage.

https://github.com/code-423n4/2022-03-timeswap/blob/main/Timeswap/Core/contracts/TimeswapPair.sol#L62

State memory state = pools[maturity].state;
