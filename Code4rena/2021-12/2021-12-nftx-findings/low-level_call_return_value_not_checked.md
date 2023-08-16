## Tags

- bug
- 2 (Med Risk)
- resolved
- sponsor confirmed

# [Low-level call return value not checked](https://github.com/code-423n4/2021-12-nftx-findings/issues/140) 

# Handle

cmichel


# Vulnerability details

The `NFTXStakingZap.addLiquidity721ETHTo` function performs a low-level `.call` in `payable(to).call{value: msg.value-amountEth}` but does not check the return value if the call succeeded.

## Impact
If the call fails, the refunds did not succeed and the caller will lose all refunds of `msg.value - amountEth`.

## Recommended Mitigation Steps
Revert the entire transaction if the refund call fails by checking that the `success` return value of the `payable(to).call(...)` returns `true`.

