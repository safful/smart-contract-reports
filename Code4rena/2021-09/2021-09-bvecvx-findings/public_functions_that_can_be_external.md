## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [public functions that can be external](https://github.com/code-423n4/2021-09-bvecvx-findings/issues/29) 

# Handle

pauliax


# Vulnerability details

## Impact
functions setWithdrawalSafetyCheck, setHarvestOnRebalance, setProcessLocksOnReinvest, and setProcessLocksOnRebalance are public but can be external as they are only supposed to be invoked from the outside.


