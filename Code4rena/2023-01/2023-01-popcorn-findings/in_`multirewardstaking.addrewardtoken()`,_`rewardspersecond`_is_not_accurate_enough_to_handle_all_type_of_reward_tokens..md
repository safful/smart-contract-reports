## Tags

- bug
- 2 (Med Risk)
- selected for report
- sponsor confirmed
- M-06

# [In `MultiRewardStaking.addRewardToken()`, `rewardsPerSecond` is not accurate enough to handle all type of reward tokens.](https://github.com/code-423n4/2023-01-popcorn-findings/issues/619) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/2d373f8748a140f5d7a57e738172750811449c04/src/utils/MultiRewardStaking.sol#L245


# Vulnerability details

## Impact
The raw `rewardsPerSecond` might be too big for some ERC20 tokens and it wouldn't work as intended.

## Proof of Concept
As we can see from `_accrueStatic()`, the `rewardsPerSecond` is a raw amount without any multiplier.

```solidity
  function _accrueStatic(RewardInfo memory rewards) internal view returns (uint256 accrued) {
    uint256 elapsed;
    if (rewards.rewardsEndTimestamp > block.timestamp) {
      elapsed = block.timestamp - rewards.lastUpdatedTimestamp;
    } else if (rewards.rewardsEndTimestamp > rewards.lastUpdatedTimestamp) {
      elapsed = rewards.rewardsEndTimestamp - rewards.lastUpdatedTimestamp;
    }

    accrued = uint256(rewards.rewardsPerSecond * elapsed);
  }
```

But 1 wei for 1 second would be too big amount for some popular tokens like WBTC(8 decimals) and EURS(2 decimals).

For WBTC, it will be 0.31536 WBTC per year (worth about `$7,200`) to meet this requirement, and for EURS, it must be at least 315,360 EURS per year (worth about `$315,000`).

Such amounts might be too big as rewards and the protocol wouldn't work properly for such tokens.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Recommend introducing a `RATE_DECIMALS_MULTIPLIER = 10**9(example)` to increase the precision of `rewardsPerSecond` instead of using the raw amount.