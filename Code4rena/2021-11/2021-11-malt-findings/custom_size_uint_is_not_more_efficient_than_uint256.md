## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Custom size uint is not more efficient than uint256](https://github.com/code-423n4/2021-11-malt-findings/issues/289) 

# Handle

0x0x0x


# Vulnerability details

## Concept

In `MovingAverage.sol`, `uint64` is used for computation of time etc. But computations with `uint64` does cost more gas and furthermore `block.timestamp` is `uint256`, which is additionally casted to `uint64`. `uint32` is used for indexes, but this can also be changed with `uint256`.

Same applies for `RewardDistributer.sol.`

## Recommendation

Use `uint256` rather than custom `uint`.

