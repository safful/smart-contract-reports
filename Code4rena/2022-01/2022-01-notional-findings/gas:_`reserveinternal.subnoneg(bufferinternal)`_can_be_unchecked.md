## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: `reserveInternal.subNoNeg(bufferInternal)` can be unchecked](https://github.com/code-423n4/2022-01-notional-findings/issues/199) 

# Handle

cmichel


# Vulnerability details

The `reserveInternal.subNoNeg(bufferInternal)` computation in `TreasuryAction.transferReserveToTreasury` can be a standard, unchecked subtraction as `if (reserveInternal <= bufferInternal) continue;` is checked before this computation.


