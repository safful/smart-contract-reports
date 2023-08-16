## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Removing the unnecessary function](https://github.com/code-423n4/2021-11-yaxis-findings/issues/96) 

# Handle

xxxxx


# Vulnerability details

## Impact
The unnecessary code can be removed to reduce contract size.

## Proof of Concept
In the contract "Alchemist.sol" the function "_expectCaller" is never used. 

## Tools Used
Remix solidity 0.6.12
## Recommended Mitigation Steps
The function "_expectCaller(address _expectedCaller)" can be removed.

