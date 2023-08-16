## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: `ConcentratedLiquidityPoolManager.addIncentive` ](https://github.com/code-423n4/2021-09-sushitrident-2-findings/issues/49) 

# Handle

cmichel


# Vulnerability details

The `ConcentratedLiquidityPoolManager.addIncentive` performs an unnecessary check:

```solidity
require(current <= incentive.endTime, "ALREADY_ENDED");
```

As it already checks that `current <= incentive.startTime` and `incentive.startTime < incentive.endTime`, this check is unnecessary and will always be true by transitivity.

## Recommended Mitigation Steps
Remove the check to save on gas.

