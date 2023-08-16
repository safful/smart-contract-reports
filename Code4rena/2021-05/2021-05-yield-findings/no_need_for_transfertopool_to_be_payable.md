## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [no need for transferToPool to be payable](https://github.com/code-423n4/2021-05-yield-findings/issues/36) 

# Handle

pauliax


# Vulnerability details

## Impact
function transferToPool is marked as 'payable'. It only transfers ERC20 tokens, no Ether, so there is no need in having 'payable' here.

## Recommended Mitigation Steps
Remove 'payable' modifier from function transferToPool.

