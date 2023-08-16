## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed
- disagree with severity

# [Missing input validation in addLiquidityForMember()](https://github.com/code-423n4/2021-07-spartan-findings/issues/225) 

# Handle

0xsanson


# Vulnerability details

## Impact
In Router.sol, the function addLiquidityForMember() doesn't check inputBase and inputToken. Since we know they can't both be zero (it wouldn't change anything and user pays the gas for nothing).

## Proof of Concept
https://github.com/code-423n4/2021-07-spartan/blob/main/contracts/Router.sol#L51

## Tools Used
editor

## Recommended Mitigation Steps
Consider adding a require `inputBase>0 || inputToken>0`.

