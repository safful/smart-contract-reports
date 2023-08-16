## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Gas: Tautology on "variable >= 0" which is always true as variable is uint](https://github.com/code-423n4/2022-01-openleverage-findings/issues/132) 

# Handle

Dravee


# Vulnerability details

## Impact  
Increased gas cost, as a variable of type `uint` will always be `>= 0`, therefore the check isn't necessary.
  
## Proof of Concept  
```
contracts\XOLE.sol:327:        require(_locked.amount >= 0, "Nothing to withdraw");
``` 

## Tools Used  
VS Code  
  
## Recommended Mitigation Steps  
Delete the `>= 0` check

