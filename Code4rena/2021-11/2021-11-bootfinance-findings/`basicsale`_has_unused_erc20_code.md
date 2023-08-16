## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [`BasicSale` has unused ERC20 code](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/210) 

# Handle

cmichel


# Vulnerability details

The `BasicSale` contract includes ERC20 code like `_balances`, `_allowances` storage variables and `Transfer`, `Approval` events.
This code is never used.

## Impact
Unused code can hint at programming or architectural errors.

## Recommended Mitigation Steps
Use it or remove it.

