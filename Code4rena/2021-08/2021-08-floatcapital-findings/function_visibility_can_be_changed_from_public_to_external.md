## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- disagree with severity
- fixed-in-upstream-repo

# [Function visibility can be changed from public to external](https://github.com/code-423n4/2021-08-floatcapital-findings/issues/87) 

# Handle

0xRajeev


# Vulnerability details

## Impact

Changing a function’s visibility from public to external can save gas by avoiding the unnecessary copying of data to memory. Function stakeFromUser() in Staker.sol is only called from SyntheticTokens.sol and not from within the contract itself which means this can be made external.


## Proof of Concept

https://github.com/code-423n4/2021-08-floatcapital/blob/bd419abf68e775103df6e40d8f0e8d40156c2f81/contracts/contracts/Staker.sol#L839

https://github.com/code-423n4/2021-08-floatcapital/blob/bd419abf68e775103df6e40d8f0e8d40156c2f81/contracts/contracts/SyntheticToken.sol#L56


## Tools Used

Manual Analysis

## Recommended Mitigation Steps

Change function visibility to external where possible.

