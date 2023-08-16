## Tags

- bug
- 1 (Low Risk)
- SwappableYieldSource
- sponsor confirmed

# [SwappableYieldSource.sol: Wrong reporting amount in FundsTransferred() event](https://github.com/code-423n4/2021-07-pooltogether-findings/issues/28) 

# Handle

hickuphh3


# Vulnerability details

### Impact

The `FundsTransferred()` event in `_transferFunds()` will report a smaller amount than expected if `currentBalance > _amount`.

This would affect applications utilizing event logs like subgraphs.

### Recommended Mitigation Steps

Update the event emission to `emit FundsTransferred(_yieldSource, currentBalance);`

