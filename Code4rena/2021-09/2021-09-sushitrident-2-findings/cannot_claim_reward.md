## Tags

- bug
- duplicate
- 2 (Med Risk)
- sponsor confirmed

# [Cannot claim reward](https://github.com/code-423n4/2021-09-sushitrident-2-findings/issues/41) 

# Handle

cmichel


# Vulnerability details

The `ConcentratedLiquidityPoolManager.claimReward` requires `stake.initialized` but it is never set.
It also performs a strange computation as `128 - incentive.secondsClaimed` which will almost always underflow and revert the transaction.

## Impact
One cannot claim rewards.

## Recommended Mitigation Steps
Rethink how claiming rewards should work.


