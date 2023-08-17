## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-08

# [CVXStaker.sol Unable to process newly add rewardTokens](https://github.com/code-423n4/2023-05-xeth-findings/issues/8) 

# Lines of code

https://github.com/code-423n4/2023-05-xeth/blob/d86fe0a9959c2b43c62716240d981ae95224e49e/src/CVXStaker.sol#L191-L196


# Vulnerability details

## Impact
The lack of a mechanism to modify `rewardTokens[]`
If `convex` adds new `extraRewards`
`CVXStaker.sol` cannot transfer the added token

## Proof of Concept
`CVXStaker.sol` will pass in `rewardTokens[]` in `constructor`
and in `getReward()`, loop this array to transfer `rewardTokens`

```solidity
    function getReward(bool claimExtras) external {
        IBaseRewardPool(cvxPoolInfo.rewards).getReward(
            address(this),
            claimExtras
        );
        if (rewardsRecipient != address(0)) {
            for (uint i = 0; i < rewardTokens.length; i++) { //<--------@audit loop, then tranfer out
                uint256 balance = IERC20(rewardTokens[i]).balanceOf(
                    address(this)
                );
                IERC20(rewardTokens[i]).safeTransfer(rewardsRecipient, balance);
            }
        }
    }
```

The main problem is that this `rewardTokens[]` does not provide a way to modify it
But it is possible to add a new `rewardsToken` in `convex`

The following code is from `BaseRewardPool.sol` of `convex`

https://github.com/convex-eth/platform/blob/main/contracts/contracts/BaseRewardPool.sol#L238

```solidity
    function addExtraReward(address _reward) external returns(bool){
        require(msg.sender == rewardManager, "!authorized");
        require(_reward != address(0),"!reward setting");

        extraRewards.push(_reward);
        return true;
    }
```


This will result in a situation : if new `extraRewards` are added to `IBaseRewardPool` later on
But since the `rewardTokens` of `CVXStaker` cannot be modified (e.g. added), then the new `extraRewards` cannot be transferred out of `CVXStaker`.
After `IBaseRewardPool(cvxPoolInfo.rewards).getReward()`, the newly added token can only stay in the `CVXStaker` contract.

## Tools Used

## Recommended Mitigation Steps

Add a new method to modify`CVXStaker.rewardTokens[]`


## Assessed type

Context