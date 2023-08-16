## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- disagree with severity

# [Wrong min amount check in `withdrawByStablecoin`](https://github.com/code-423n4/2021-06-gro-findings/issues/97) 

# Handle

cmichel


# Vulnerability details

## Vulnerability Details
The `WithdrawHandler.withdrawByStablecoin` incorrectly uses the `lpAmount` instead of the `minAmount` in the check.

```solidity
require(lpAmount > 0, "!minAmount");
```

## Recommended Mitigation Steps
Use `minAmount > 0` if trying to check for `!minAmount` or use a different error message for an invalid LP amount.

