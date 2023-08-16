## Tags

- bug
- 1 (Low Risk)
- mStableYieldSource
- sponsor confirmed

# [Validation](https://github.com/code-423n4/2021-07-pooltogether-findings/issues/74) 

# Handle

pauliax


# Vulnerability details

## Impact
function supplyTokenTo should check that mAssetAmount and creditsIssued > 0 and to != address(0) or if empty to address is provided, it can replace it with msg.sender to prevent potential burn of funds. function redeemToken should check that mAssetAmount and creditsBurned > 0. function transferERC20 should similarly validate erc20Token, to and amount parameters. function _mintShares requires that shares > 0, while _burnShares lacks such requirement.


