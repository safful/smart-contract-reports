## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-11

# [StandardFunding.fundingVote should not allow users who didn't vote in screening stage to vote](https://github.com/code-423n4/2023-05-ajna-findings/issues/224) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/StandardFunding.sol#L519-L569
https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/StandardFunding.sol#L519


# Vulnerability details

## Impact

Users who did not vote in the screening stage but voted in the funding stage are not allowed to claim rewards via `claimDelegateReward`. **Voting in the funding stage will occupy the distribution ratio of rewards**. Since these rewards cannot be claimed, in the long run, the ajnaToken balance of the GrantFund contract is inconsistent with `treasury`.

## Proof of Concept

At the beginning of the `StandardFunding.claimDelegateReward` function, check whether the caller voted in the screening stage.

```solidity
function claimDelegateReward(
        uint24 distributionId_
    ) external override returns(uint256 rewardClaimed_) {
        // Revert if delegatee didn't vote in screening stage
->      if(screeningVotesCast[distributionId_][msg.sender] == 0) revert DelegateRewardInvalid();

        QuarterlyDistribution memory currentDistribution = _distributions[distributionId_];
        ...
    }
```

`StandardFunding.fundingVote` is used to vote in the funding stage. This function does not check whether the caller voted in the screening stage. `fundingVote` subcalls [_fundingVote](https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/StandardFunding.sol#L612), which affects the allocation of rewards. `_getDelegateReward` is used by `claimDelegateReward` to calculate the reward distributed to the caller.

```solidity
function _getDelegateReward(
        QuarterlyDistribution memory currentDistribution_,
        QuadraticVoter memory voter_
    ) internal pure returns (uint256 rewards_) {
        // calculate the total voting power available to the voter that was allocated in the funding stage
        uint256 votingPowerAllocatedByDelegatee = voter_.votingPower - voter_.remainingVotingPower;

        // if none of the voter's voting power was allocated, they receive no rewards
        if (votingPowerAllocatedByDelegatee == 0) return 0;

        // calculate reward
        // delegateeReward = 10 % of GBC distributed as per delegatee Voting power allocated
->      rewards_ = Maths.wdiv(
            Maths.wmul(
                currentDistribution_.fundsAvailable,	//total funds in current distribution
                votingPowerAllocatedByDelegatee		//voter's vote power
            ),
            currentDistribution_.fundingVotePowerCast	//total vote power in current distribution
        ) / 10;						// 10% fundsAvailable
    }
```

As long as `fundingVote` is successfully called, it means that **the reward is locked to the caller**. However, the caller cannot claim these rewards. **There is no code to calculate the amount of these rewards that can never be claimed**.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Two ways to fix this problem:

1.  `FundingVote` does not allow users who didn't vote in screening stage.
2.  Delete the code on line [L240](https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/StandardFunding.sol#L240).


## Assessed type

Other