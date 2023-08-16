## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [`increaseApproval` gas improval](https://github.com/code-423n4/2021-07-sherlock-findings/issues/98) 

# Handle

cmichel


# Vulnerability details

The `SherXERC20.increaseApproval` function reads the allowance from memory twice.
It should be read once and then cached and for the event you the `cache + _amount` value.

