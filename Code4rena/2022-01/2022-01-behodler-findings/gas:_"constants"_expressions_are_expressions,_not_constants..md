## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Gas: "constants" expressions are expressions, not constants.](https://github.com/code-423n4/2022-01-behodler-findings/issues/197) 

# Handle

Dravee


# Vulnerability details

## Impact  
Due to how `constant` variables are implemented (replacements at compile-time), an expression assigned to a `constant` variable is recomputed each time that the variable is used, which wastes some gas.  
  
See: [ethereum/solidity#9232](https://github.com/ethereum/solidity/issues/9232) 
> Consequences: each usage of a "constant" costs ~100gas more on each access (it is still a little better than storing the result in storage, but not much..). since these are not real constants, they can't be referenced from a real constant environment (e.g. from assembly, or from another library )
  
## Proof of Concept  
```
UniswapHelper.sol:56:  uint256 constant year = (1 days * 365);
``` 
  
## Tools Used  
VS Code  
  
## Recommended Mitigation Steps  
Replace with:
```
UniswapHelper.sol:56:  uint256 constant year = 365 days;
``` 


