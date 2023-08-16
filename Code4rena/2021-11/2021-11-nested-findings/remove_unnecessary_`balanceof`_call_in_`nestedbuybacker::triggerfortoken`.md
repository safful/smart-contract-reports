## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Remove unnecessary `balanceOf` call in `NestedBuybacker::triggerForToken`](https://github.com/code-423n4/2021-11-nested-findings/issues/65) 

# Handle

pmerkleplant


# Vulnerability details

Function `triggerForToken` in `NestedBuybacker.sol` makes a `balanceOf` call on
the `_sellToken`, see [line 100](https://github.com/code-423n4/2021-11-nested/blob/main/contracts/NestedBuybacker.sol#L100).

However, the result of the call is never used.

It would save gas to remove the unnecessary call and variable declaration.

