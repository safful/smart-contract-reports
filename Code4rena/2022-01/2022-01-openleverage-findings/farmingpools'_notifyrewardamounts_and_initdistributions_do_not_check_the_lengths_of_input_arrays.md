## Tags

- bug
- 0 (Non-critical)
- resolved
- sponsor confirmed

# [FarmingPools' notifyRewardAmounts and initDistributions do not check the lengths of input arrays](https://github.com/code-423n4/2022-01-openleverage-findings/issues/76) 

# Handle

hyh


# Vulnerability details


## Impact

On calling with arrays of different lengths various malfunctions are possible as the arrays are used as given. System then will fail with low level array access message

## Proof of Concept

notifyRewardAmounts:

https://github.com/code-423n4/2022-01-openleverage/blob/main/openleverage-contracts/contracts/farming/FarmingPools.sol#L163


initDistributions:

https://github.com/code-423n4/2022-01-openleverage/blob/main/openleverage-contracts/contracts/farming/FarmingPools.sol#L131

## Recommended Mitigation Steps

Require that (stakeTokens, reward) and (stakeTokens, startTimes, durations) arrays' lengths match within each set


