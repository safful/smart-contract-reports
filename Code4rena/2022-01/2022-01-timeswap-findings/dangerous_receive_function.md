## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [dangerous receive function](https://github.com/code-423n4/2022-01-timeswap-findings/issues/35) 

# Handle

danb


# Vulnerability details

https://github.com/code-423n4/2022-01-timeswap/blob/main/Timeswap/Timeswap-V1-Convenience/contracts/TimeswapConvenience.sol#L69

the contract should receive ether only from weth,

consider adding:

```
require(msg.sender == weth);
```

