## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [payable vest](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/273) 

# Handle

pauliax


# Vulnerability details

## Impact
There is no reason for the function vest to be 'payable' as it does not handle ether in any way and there is no way to rescue it later in case someone accidentally sends it.

## Recommended Mitigation Steps
Remove 'payable' from the vest function.

