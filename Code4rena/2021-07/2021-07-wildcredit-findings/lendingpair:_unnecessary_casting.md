## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [LendingPair: Unnecessary Casting](https://github.com/code-423n4/2021-07-wildcredit-findings/issues/59) 

# Handle

greiart


# Vulnerability details

## Impact

The address casting of `_token` in `lpToken[address(_token)]` can be removed in the `withdrawAll()`, `_withdraw()` and `_borrow()` functions, since `_token` is already an address in these functions. 

In other words, `lpToken[address(_token)]` *→* `lpToken[token]`

