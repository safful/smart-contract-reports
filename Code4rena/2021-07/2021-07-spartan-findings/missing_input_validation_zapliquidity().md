## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Missing input validation zapLiquidity()](https://github.com/code-423n4/2021-07-spartan-findings/issues/222) 

# Handle

0xsanson


# Vulnerability details

## Impact
zapLiquidity() in Router.sol misses an input validation unitsInput > 0.

## Proof of Concept
https://github.com/code-423n4/2021-07-spartan/blob/main/contracts/Router.sol#L59

## Tools Used
editor

## Recommended Mitigation Steps
Add an input validation for unitsInput.

