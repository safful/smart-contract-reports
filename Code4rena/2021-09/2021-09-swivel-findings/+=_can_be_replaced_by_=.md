## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [+= can be replaced by =](https://github.com/code-423n4/2021-09-swivel-findings/issues/113) 

# Handle

0xRajeev


# Vulnerability details

## Impact

to.notional += a  can be replaced by to.notional = a because to.notional = 0 in the else part. This will save a few MLOADs.

## Proof of Concept

https://github.com/Swivel-Finance/gost/blob/5fb7ad62f1f3a962c7bf5348560fe88de0618bae/test/vaulttracker/VaultTracker.sol#L189

## Tools Used
Manual Analysis

## Recommended Mitigation Steps

Replace to.notional += a  by to.notional = a

