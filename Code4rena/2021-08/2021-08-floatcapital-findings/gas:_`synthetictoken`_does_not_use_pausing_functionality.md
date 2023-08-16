## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: `SyntheticToken` does not use pausing functionality](https://github.com/code-423n4/2021-08-floatcapital-findings/issues/118) 

# Handle

cmichel


# Vulnerability details

The `SyntheticToken` overwrites the `_beforeTokenTransfer` hook and removes the pausing functionality of `ERC20PresetMinterPauser`.
But the `ERC20PresetMinterPauser` constructor still assigns pauser roles which leads to unnecessary gas costs.
Inherit from an `ERC20PresetMinterPauser`-like contract without the pausing functionality.
This would also make the intention of the code more clear by showcasing that it does not implement the pauser interface functions `pause`/`unpause` (which it currently still does but they don't have any effect).


