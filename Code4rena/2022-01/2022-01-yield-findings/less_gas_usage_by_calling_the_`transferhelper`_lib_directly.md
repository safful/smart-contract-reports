## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [less gas usage by calling the `TransferHelper` lib directly](https://github.com/code-423n4/2022-01-yield-findings/issues/72) 

# Handle

rfa


# Vulnerability details

## Impact
spend at least 6930 more gas on deployment, and spend 40 gas more per call (by using current implementasion)

## Proof of Concept
https://github.com/code-423n4/2022-01-yield/blob/main/contracts/ConvexStakingWrapper.sol#L184
https://github.com/code-423n4/2022-01-yield/blob/main/contracts/ConvexStakingWrapper.sol#L239

the `TransferHelper` lib just used twice in this contract.
remove:(line 16) https://github.com/code-423n4/2022-01-yield/blob/main/contracts/ConvexStakingWrapper.sol#L16

and just call `TransferHelper.safeTransfer()` directly at those line.

This method is using almost exact the same gas as if we just copying the `safeTransfer()` and remove the `TransferHelper` lib from the contract. (since we need just 1 function from the lib)



