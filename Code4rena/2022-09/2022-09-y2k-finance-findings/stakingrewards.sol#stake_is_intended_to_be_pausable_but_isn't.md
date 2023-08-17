## Tags

- bug
- 2 (Med Risk)
- sponsor disputed
- selected for report

# [StakingRewards.sol#stake is intended to be pausable but isn't](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/38) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/ac3e86f07bc2f1f51148d2265cc897e8b494adf7/src/rewards/StakingRewards.sol#L90-L107


# Vulnerability details

## Impact

Staking is unable to be paused as intended

## Proof of Concept

StakingRewards.sol inherits pausable and implements the whenNotPaused modifier on stake, but doesn't implement any method to actually pause or unpause the contract. Pausable.sol only implements internal functions, which requires external or public functions to be implemented to wrap them. Since nothing like this has been implemented, the entire pausing system is rendered useless and staking cannot be paused as is intended.

## Tools Used

## Recommended Mitigation Steps

Create simple external pause and unpause functions that can be called by owner