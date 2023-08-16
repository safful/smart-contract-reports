## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed

# [UserManager: totalStaked ≥ totalFrozen should be checked before and after totalFrozen is updated](https://github.com/code-423n4/2021-10-union-findings/issues/47) 

# Handle

itsmeSTYJ


# Vulnerability details

## Impact

The require statement in `updateTotalFrozen` and `batchUpdateTotalFrozen` to check that totalStaked ≥ totalFrozen should be done both before and after `_updateTotalFrozen` is called to ensure that totalStake is still ≥ totalFrozen. This will serve as a sanity check to ensure that the integrity of the system is not compromised.

