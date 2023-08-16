## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [MAX_TOTAL_XDEFI_SUPPLY should be constant](https://github.com/code-423n4/2022-01-xdefi-findings/issues/36) 

# Handle

agusduha


# Vulnerability details

## Impact

MAX_TOTAL_XDEFI_SUPPLY has always the same value and is used only in one place, it should be constant to optimize gas

## Proof of Concept

Variable declaration: https://github.com/XDeFi-tech/xdefi-distribution/blob/3856a42df295183b40c6eee89307308f196612fe/contracts/XDEFIDistribution.sol#L14

Variable utilization: https://github.com/XDeFi-tech/xdefi-distribution/blob/3856a42df295183b40c6eee89307308f196612fe/contracts/XDEFIDistribution.sol#L255

## Tools Used

Manual analysis

## Recommended Mitigation Steps

Add the "constant" keyword to the storage variable declaration

