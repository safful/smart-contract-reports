## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: `transferGovernance` can save an sload](https://github.com/code-423n4/2021-10-tracer-findings/issues/16) 

# Handle

cmichel


# Vulnerability details

The `LeveragedPool.transferGovernance` function emits an event and reads the new governance variable from storage.

```solidity
emit ProvisionalGovernanceChanged(provisionalGovernance);
```

It is cheaper to use the `_governance` parameter instead which is the same value.


