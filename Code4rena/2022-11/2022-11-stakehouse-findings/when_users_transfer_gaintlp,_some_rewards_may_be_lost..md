## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- judge review requested
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-19

# [When users transfer GaintLP, some rewards may be lost.](https://github.com/code-423n4/2022-11-stakehouse-findings/issues/238) 

# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/GiantMevAndFeesPool.sol#L146-L148


# Vulnerability details

## Impact
GiantMevAndFeesPool.beforeTokenTransfer will try to distribute the user's current rewards to the user when transferring GaintLP, but since beforeTokenTransfer will not call StakingFundsVault.claimRewards to claim the latest rewards, thus making the calculated accumulatedETHPerLPShare smaller and causing the user to lose some rewards.
## Proof of Concept
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/GiantMevAndFeesPool.sol#L146-L148
## Tools Used
None
## Recommended Mitigation Steps
Consider claiming the latest rewards from StakingFundsVault before the GiantMevAndFeesPool.beforeTokenTransfer calls updateAccumulatedETHPerLP()