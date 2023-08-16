## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [`initializeSherXERC20` can be called more than once](https://github.com/code-423n4/2021-07-sherlock-findings/issues/116) 

# Handle

cmichel


# Vulnerability details

The `SherXERC20.initializeSherXERC20` function has `initialize` in its name which indicates that it should only be called once to initialize the storage. But it can be repeatedly called to overwrite and update the ERC20 name and symbol.

## Recommendation
Consider an `initializer` modifier or reverting if `name` or `symbol` is already set.

