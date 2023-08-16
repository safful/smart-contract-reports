## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [No used library added](https://github.com/code-423n4/2021-11-nested-findings/issues/114) 

# Handle

xYrYuYx


# Vulnerability details

## Impact
https://github.com/code-423n4/2021-11-nested/blob/main/contracts/NestedBuybacker.sol#L15

There is only NST.safeTransfer used, and NST is INestedToken interface.
SafeERC20 is not used for IERC20 interface.

## Tools Used

## Recommended Mitigation Steps
Remove Line 79

