## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- YearnV2YieldSource

# [Using function parameter in initialize() instead of state variable saves 100 gas](https://github.com/code-423n4/2021-06-pooltogether-findings/issues/43) 

# Handle

0xRajeev


# Vulnerability details

## Impact

Using parameter _vault instead of SLOAD of state variable vault in the call to safeApprove() leads to gas savings of 100.

## Proof of Concept

https://github.com/code-423n4/2021-06-pooltogether/blob/85f8d044e7e46b7a3c64465dcd5dffa9d70e4a3e/contracts/yield-source/YearnV2YieldSource.sol#L87

https://github.com/code-423n4/2021-06-pooltogether/blob/85f8d044e7e46b7a3c64465dcd5dffa9d70e4a3e/contracts/yield-source/YearnV2YieldSource.sol#L67

https://github.com/code-423n4/2021-06-pooltogether/blob/85f8d044e7e46b7a3c64465dcd5dffa9d70e4a3e/contracts/yield-source/YearnV2YieldSource.sol#L25


## Tools Used

Manual Analysis

## Recommended Mitigation Steps

Using parameter _vault instead of state variable vault in the call to safeApprove()

