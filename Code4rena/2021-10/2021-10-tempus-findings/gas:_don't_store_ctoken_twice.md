## Tags

- bug
- sponsor confirmed
- G (Gas Optimization)
- resolved

# [Gas: Don't store cToken twice](https://github.com/code-423n4/2021-10-tempus-findings/issues/26) 

# Handle

cmichel


# Vulnerability details

In `CompoundTempusPool`, the `cToken` and the base class' `yieldBearingToken` storage fields are the same.
Remove the `cToken` field and the assignment in the constructor to save gas.


