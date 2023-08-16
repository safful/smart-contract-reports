## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- resolved

# [Inconsistent NatSpec comment in Pool.sol](https://github.com/code-423n4/2021-04-maple-findings/issues/71) 

# Handle

0xRajeev


# Vulnerability details

## Impact

The isValidDelegateorAdmin() is used for access control on both setLiquidityCap() and claim() but the @dev Natspec comment only specifies setLiquidityCap() which is misleading.

## Proof of Concept

https://github.com/maple-labs/maple-core/blob/355141befa89c7623150a83b7d56a5f5820819e9/contracts/Pool.sol#L597

## Tools Used

Manual Analysis

## Recommended Mitigation Steps

Add claim() as well to @dev on L597.


