## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Unused state variables](https://github.com/code-423n4/2022-01-notional-findings/issues/204) 

# Handle

pauliax


# Vulnerability details

## Impact
Unused state variables:
```solidity
    uint256 public constant BPT_TOKEN_PRECISION = 1e18;
    uint256 internal constant ETH_PRECISION = 1e18;
    uint32 public refundGasPrice;
```
Either remove them or use them where intended.

