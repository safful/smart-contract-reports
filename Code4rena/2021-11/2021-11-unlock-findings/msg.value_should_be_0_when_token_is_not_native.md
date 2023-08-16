## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [msg.value should be 0 when token is not native](https://github.com/code-423n4/2021-11-unlock-findings/issues/220) 

# Handle

pauliax


# Vulnerability details

## Impact
function purchase is payable, thus it should validate that msg.value is 0 when tokenAddress != address(0) to prevent accidental sent Ether.

## Recommended Mitigation Steps
Check no ether was sent when the token is not a native currency.

