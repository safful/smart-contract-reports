## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [StrategyCvxHelper: safeApprove instead of approve](https://github.com/code-423n4/2021-09-bvecvx-findings/issues/33) 

# Handle

hickuphh3


# Vulnerability details

### Impact

This was probably an oversight since

- the veCVXStrategy contract used `safeApprove()` for token approvals
- `using SafeERC20Upgradeable for IERC20Upgradeable;` was declared

### Recommended Mitigation Steps

Change

`cvxToken.approve(address(cvxRewardsPool), MAX_UINT_256);`

to

`cvxToken.safeApprove(address(cvxRewardsPool), MAX_UINT_256);`

