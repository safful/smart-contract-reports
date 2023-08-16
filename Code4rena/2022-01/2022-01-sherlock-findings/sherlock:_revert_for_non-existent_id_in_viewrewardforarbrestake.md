## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [Sherlock: Revert for non-existent ID in viewRewardForArbRestake](https://github.com/code-423n4/2022-01-sherlock-findings/issues/225) 

# Handle

GreyArt


# Vulnerability details

## Impact

Other relevant view functions like `lockupEnd()`, `sherRewards()` and `tokenBalanceOf()` revert for non-existent IDs, but `viewRewardForArbRestake()` doesn’t.

## Recommended Mitigation Steps

Include the existence check in `viewRewardForArbRestake()`.

`if (!_exists(_tokenID)) revert NonExistent();`

