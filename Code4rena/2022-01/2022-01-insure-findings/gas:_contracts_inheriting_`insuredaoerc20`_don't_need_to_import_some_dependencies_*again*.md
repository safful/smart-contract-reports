## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Gas: Contracts inheriting `InsureDAOERC20` don't need to import some dependencies *again*](https://github.com/code-423n4/2022-01-insure-findings/issues/78) 

# Handle

Dravee


# Vulnerability details

## Impact
When a contract imports and implements an interface or another contracts, it doesn't need to import the libraries that were already imported there.

Removing these imports will save gas.

## Proof of Concept
`InsureDAOERC20` imports the following: 
```
5: import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
6: import "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
```

The following contracts inherit `InsureDAOERC20` and also make those imports: `CDSTemplate`, `IndexTemplate`, `PoolTemplate`

## Tools Used
VS Code

## Recommended Mitigation Steps
Remove the unused imports to reduce the size of the contract and save some deployment gas.

