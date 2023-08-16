## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Gas: Cache `attributions[_target]` in `Vault.sol:underlyingValue()`](https://github.com/code-423n4/2022-01-insure-findings/issues/343) 

# Handle

Dravee


# Vulnerability details

## Impact
SLOADs are expensive

## Proof of Concept
Here, `attributions[_target]` can be loaded twice from storage:
```
Vault.sol
400:     function underlyingValue(address _target)
401:         public
402:         view
403:         override
404:         returns (uint256)
405:     {
406:         if (attributions[_target] > 0) {
407:             return (valueAll() * attributions[_target]) / totalAttributions;
408:         } else {
409:             return 0;
410:         }
411:     }
```

## Tools Used
VS Code

## Recommended Mitigation Steps
Cache the loaded storage value in a memory variable

