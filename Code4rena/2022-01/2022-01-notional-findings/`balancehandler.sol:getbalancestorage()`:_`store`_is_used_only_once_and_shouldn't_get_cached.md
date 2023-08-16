## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [`BalanceHandler.sol:getBalanceStorage()`: `store` is used only once and shouldn't get cached](https://github.com/code-423n4/2022-01-notional-findings/issues/125) 

# Handle

Dravee


# Vulnerability details

## Impact
Increased gas cost

## Proof of Concept
`store` is a variable used only once. A comment should suffice instead of a variable (see @audit-info):
```
File: BalanceHandler.sol
72:         mapping(address => mapping(uint256 => BalanceStorage)) storage store = LibStorage.getBalanceStorage(); //@audit-info store is used only once, below
73:         BalanceStorage storage balanceStorage = store[account][currencyId];
```
## Tools Used
VS Code

## Recommended Mitigation Steps
Do not store this data in a variable. Inline it instead:
```
BalanceStorage storage balanceStorage = LibStorage.getBalanceStorage()[account][currencyId];
```



