## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [PoolTemplate.availableBalance calls totalLiquidity twice](https://github.com/code-423n4/2022-01-insure-findings/issues/262) 

# Handle

hyh


# Vulnerability details

## Impact

Gas is overspent on the function call

## Proof of Concept

availableBalance calls totalLiquidity() twice:

https://github.com/code-423n4/2022-01-insure/blob/main/contracts/PoolTemplate.sol#L835

## Recommended Mitigation Steps

Save the call result to memory and use it

