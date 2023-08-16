## Tags

- bug
- sponsor confirmed
- 0 (Non-critical)
- resolved

# [Comment Typos](https://github.com/code-423n4/2021-10-pooltogether-findings/issues/59) 

# Handle

leastwood


# Vulnerability details

## Impact

There are a couple of typos found within the `Reserve.sol` contract.

## Proof of Concept

https://github.com/pooltogether/v4-core/blob/master/contracts/Reserve.sol#L20
https://github.com/pooltogether/v4-core/blob/master/contracts/Reserve.sol#L21

## Tools Used

Manual code review

## Recommended Mitigation Steps

Consider updating the typo in `Reserve.sol:L20` from `speicific` to `specific` and the typo in `Reserve.sol:L21` from `determininstially` to `deterministically`.

