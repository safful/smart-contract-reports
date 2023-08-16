## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [DelegatorPool.claim subtraction can be unchecked and done once](https://github.com/code-423n4/2022-01-livepeer-findings/issues/154) 

# Handle

hyh


# Vulnerability details

## Impact

Gas is overspent on calculations and checks

## Proof of Concept

(initialStake - claimedInitialStake) figure is calculated after require check, so the subtraction itself can be unchecked. Also, it is done twice now, can save the result to memory and use it.

https://github.com/livepeer/arbitrum-lpt-bridge/blob/main/contracts/L2/pool/DelegatorPool.sol#L73

## Recommended Mitigation Steps

Consider calculating (initialStake - claimedInitialStake) one time and in unchecked scope.

