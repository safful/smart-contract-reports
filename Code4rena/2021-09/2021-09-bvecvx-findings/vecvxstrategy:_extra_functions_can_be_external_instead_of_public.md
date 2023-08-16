## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [veCVXStrategy: Extra functions can be external instead of public](https://github.com/code-423n4/2021-09-bvecvx-findings/issues/36) 

# Handle

hickuphh3


# Vulnerability details

### Impact

The `setWithdrawalSafetyCheck()`, `setHarvestOnRebalance()`, `setProcessLocksOnReinvest()` and `setProcessLocksOnRebalance()` functions are unused internally but have `public` visibility. Their visibility can be changed to `external`.

