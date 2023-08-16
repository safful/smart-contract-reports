## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: Explicit overflow checks even though solidity 0.8 is used (2)](https://github.com/code-423n4/2021-10-union-findings/issues/75) 

# Handle

cmichel


# Vulnerability details

The `UToken` contract uses solidity version 0.8 which already comes with implicit overflow checks.
The explicit overflow checks in `removeReserves` can be removed:

```solidity
// We checked reduceAmount <= totalReserves above, so this should never revert.
// @audit this overflow check already happened implicitly
require(totalReservesNew <= totalReserves, "reduce reserves unexpected underflow");
```

