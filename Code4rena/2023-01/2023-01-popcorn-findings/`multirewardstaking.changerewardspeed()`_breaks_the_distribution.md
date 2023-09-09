## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- selected for report
- sponsor confirmed
- M-29

# [`MultiRewardStaking.changeRewardSpeed()` breaks the distribution](https://github.com/code-423n4/2023-01-popcorn-findings/issues/190) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L296-L315
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L351-L360


# Vulnerability details

## Impact
The `changeRewardSpeed()` doesn't calculate the new `endTimestamp` correctly. That causes the reward distribution to be broken

## Proof of Concept
Given that we have an existing reward with the following configuration:
* startTimestamp = 0
* endTimestamp = 100
* rewardPerSecond = 2
* initialBalance = 200

The reward speed is changed at `timestamp = 50`, meaning 100 tokens were already distributed. The new endTimestamp is calculated by calling `_calcRewardsEnd()`:
```sol
    // @audit using balanceOf() here has its own issues but let's ignore those for this submission
    uint256 remainder = rewardToken.balanceOf(address(this));

    uint32 prevEndTime = rewards.rewardsEndTimestamp;

    uint32 rewardsEndTimestamp = _calcRewardsEnd(
      prevEndTime > block.timestamp ? prevEndTime : block.timestamp.safeCastTo32(),
      rewardsPerSecond,
      remainder
    );
```
And the calculation is:
```sol
  function _calcRewardsEnd(
    uint32 rewardsEndTimestamp,
    uint160 rewardsPerSecond,
    uint256 amount
  ) internal returns (uint32) {
    if (rewardsEndTimestamp > block.timestamp)
      amount += uint256(rewardsPerSecond) * (rewardsEndTimestamp - block.timestamp);

    return (block.timestamp + (amount / uint256(rewardsPerSecond))).safeCastTo32();
  }

```

* `rewardsEndTimestamp = 100` (initial endTimestamp)
* `block.timestamp = 50` (as described earlier)
* `amount = 100` 
* `rewardsPerSecond = 4` (we update it by calling this function)

Because `rewardEndTimestamp > block.timestamp`, the if clause is executed and `amount` is increased:
$amountNew = 100 + 4 * (100 - 50) = 300$

Then it calculates the new `endTimestamp`:
$50 + (300 / 4) = 125$

Thus, by increasing the `rewardsPerSecond` from `2` to `4`, we've **increased** the `endTimestamp` from `100` to `125` instead of descreasing it. The total amount of rewards that are distributed are calculated using the `rewardsPerSecond` and `endTimestamp`. Meaning, the contract will also try to distribute tokens it doesn't hold. It only has the remaining `100` tokens.

By increasing the `rewardsPerSecond` the whole distribution is broken.
## Tools Used
none

## Recommended Mitigation Steps
It's not easy to fix this issue with the current implementation of the contract. There are a number of other issues. But, in essence:
* determine the remaining amount of tokens that need to be distributed
* calculate the new endTimestamp: `endTimestamp = remainingAmount / newRewardsPerSecond`