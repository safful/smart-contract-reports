## Tags

- bug
- sponsor confirmed
- 0 (Non-critical)

# [Unused constants could indicate missing logic or redundant code](https://github.com/code-423n4/2021-09-sushitrident-findings/issues/43) 

# Handle

0xRajeev


# Vulnerability details

## Impact

Constants MAX_FEE_SQUARE and E18 are declared but never used. Unused constants could indicate missing logic or redundant code. In this case, they are likely to be redundant code that can be removed.

## Proof of Concept

https://github.com/sushiswap/trident/blob/504e2e2f3929175eb7adc73844c381d5174e1c03/contracts/pool/ConstantProductPool.sol#L25

https://github.com/sushiswap/trident/blob/504e2e2f3929175eb7adc73844c381d5174e1c03/contracts/pool/ConstantProductPool.sol#L26

## Tools Used
Manual Analysis

## Recommended Mitigation Steps

Evaluate the use of the declared constants or remove them.

