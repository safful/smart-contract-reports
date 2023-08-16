## Tags

- bug
- G (Gas Optimization)
- sponsor acknowledged
- sponsor confirmed

# [VaultHelper could validate that amount is greater than 0](https://github.com/code-423n4/2021-09-yaxis-findings/issues/85) 

# Handle

pauliax


# Vulnerability details

## Impact
functions depositVault, depositMultipleVault and withdrawVault in VaultHelper could require _amount > 0 to prevent useless transfers.

## Recommended Mitigation Steps
Add require _amount > 0 statements to mentioned functions.

