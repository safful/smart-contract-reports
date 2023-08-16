## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Gov.sol: Use SafeERC20.safeApprove in tokenUnload()](https://github.com/code-423n4/2021-07-sherlock-findings/issues/51) 

# Handle

hickuphh3


# Vulnerability details

### Impact

This is probably an oversight since `SafeERC20` was imported and `safeTransfer()` was used for ERC20 token transfers. Nevertheless, note that `approve()` will fail for certain token implementations that do not return a boolean value (Eg. OMG and ADX tokens). Hence it is recommend to use `safeApprove()`.

### Recommended Mitigation Steps

Update to `_token.safeApprove(address(_native), totalToken)` in `tokenUnload()`.

