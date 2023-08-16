## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed
- disagree with severity

# [ERC20 non-standard names](https://github.com/code-423n4/2021-07-sherlock-findings/issues/117) 

# Handle

cmichel


# Vulnerability details

Usually, the functions to increase the allowance are called `increaseAllowance` and `decreaseAllowance` but in `SherXERC20` they are called `increaseApproval` and `decreaseApproval`

## Recommendation
Rename these functions to the more common names.

