## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Gas: Consider making variables that aren't updated outside the constructor as `immutable`](https://github.com/code-423n4/2022-01-insure-findings/issues/72) 

# Handle

Dravee


# Vulnerability details

## Impact
The compiler won't reserve a storage slot for `immutable` variables

## Proof of Concept
The following variables are initialized in the contract's constructor and can't get updated after:
```
Factory.sol:registry
Factory.sol:ownership
Parameters:ownership
BondingPremium:ownership
Registry:ownership
Vault:ownership
```

## Tools Used
VS Code

## Recommended Mitigation Steps
Make these variables `immutable`

