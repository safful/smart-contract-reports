## Tags

- bug
- sponsor confirmed
- G (Gas Optimization)
- resolved

# [TempusAMM freezing all actions except proportional exit on maturity seems unnecessary](https://github.com/code-423n4/2021-10-tempus-findings/issues/18) 

# Handle

TomFrench


# Vulnerability details

## Impact

Reduced flexibility of AMM + additional gas costs on swaps

## Proof of Concept

As the relative payouts of the principal and yield tokens are fixed at the point of finalisation, there's no need to freeze the AMM as it will just rapidly be arbed to the final prices of each token. No funds will be lost by LPs.

Making this change would reduce gas costs as swaps won't have to check maturity (load the TempusPool then perform SLOAD for maturity state variable).

## Recommended Mitigation Steps

Remove `beforeMaturity` modifier from AMM.

