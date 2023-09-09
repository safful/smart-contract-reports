## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-27

# [Faulty Escrow config will lock up reward tokens in Staking contract](https://github.com/code-423n4/2023-01-popcorn-findings/issues/251) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/VaultController.sol#L443-L471
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L265-L270
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L178-L179
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L201
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardEscrow.sol#L97-L98


# Vulnerability details

## Impact
When a Vault owner/create adds a reward token to the Staking contract with faulty escrow config parameters, the reward tokens sent to the Staking contract will forever be locked up. This can happen since there are no validity checks on the values of the Escrow config for the reward tokens when adding a reward token to the Staking contract via the VaultController. 

## Proof of Concept
Below are the steps to reproduce the issue:
1. Vault Creator/Owner adds a reward token to the Staking contract of a vault they own/created, passing Escrow configuration parameters of `useEscrow` = true, `escrowDuration` = 0 and `escrowPercentage` = 1. This passes without issue since there are no validity checks for the Escrow config in the following lines:
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/VaultController.sol#L443-L471
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L265-L271

2. This issue not noticeable until someone attempts to `claimRewards` for that misconfigured rewardToken from the Staking contract. This will always revert for all users trying to claim rewards for the affected rewardToken since it always attempts to lock some funds in escrow:
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L178-L180
And one of these checks in `Escrow.lock` will always fail since `escrowDuration` was set to 0 and `escrowPercentage` is so low that rewards must be so high for the escrow amount to not be 0: https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardEscrow.sol#L97-L98
3. Reward tokens for that misconfigured rewardToken contract will now forever be locked in the Staking contract leading to loss of funds vault creator/owner.


## Tools Used
VSCode

## Recommended Mitigation Steps
1. Add validity checks for escrowDuration and escrowPercentage before these lines: https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L265-L270
Make sure that `escrowDuration` is greater than 0 and that `escrowPercentage` is high enough that it won't always trigger reverts when users claim rewards.
2. If reward amounts are too small, the escrow amount will be 0 and that will revert the `claimRewards` so users will not be able to claim rewards. Maybe check if there are escrowed amount is greather than 0 and only call `Escrow.lock` if it is. That way, users with small rewards will always be able to claim funds. Note that only users will larger rewards at time of claiming will have funds escrowed for smaller escrow percentages.