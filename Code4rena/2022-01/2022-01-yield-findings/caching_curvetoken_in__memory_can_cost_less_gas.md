## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [caching curveToken in  memory can cost less gas](https://github.com/code-423n4/2022-01-yield-findings/issues/49) 

# Handle

Funen


# Vulnerability details

https://github.com/code-423n4/2022-01-yield/blob/main/contracts/ConvexStakingWrapper.sol#L94-L95

```
IERC20(curveToken).approve(convexBooster, 0);
IERC20(curveToken).approve(convexBooster, type(uint256).max);
```

`curveToken` was called mutiple times, caching it in `memory` , it can cost less gas

