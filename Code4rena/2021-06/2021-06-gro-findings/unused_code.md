## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Unused code](https://github.com/code-423n4/2021-06-gro-findings/issues/71) 

# Handle

pauliax


# Vulnerability details

## Impact
contract LifeGuard3Pool has unused events: LogHealhCheckUpdate, LogNewEmergencyWithdrawal. Interfaces IHarvest and IStake are not used. contract Buoy3Pool has unused variable TIME_LIMIT and a variable that is only initialized but never used: lpToken. 
Style issue: BASIS_POINTS all caps indicate it should be a constant, however, an owner can change it by calling function setBasisPointsLmit.

## Recommended Mitigation Steps
Make use of this code or remove it. 

