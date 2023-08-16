## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Use of floating pragmas](https://github.com/code-423n4/2021-12-amun-findings/issues/6) 

# Handle

jayjonah8


# Vulnerability details

## Impact
Floating pragmas are used throughout the codebase.  Contracts should be deployed with the same compiler version that they have been tested with thoroughly. Locking the pragma helps to ensure that contracts do not accidentally get deployed using, for example, an outdated or different compiler version that might introduce bugs that affect the contract system negatively.

## Proof of Concept
https://swcregistry.io/docs/SWC-103

Floating pragmas are used throughout the protocol 

## Tools Used
Manual code review 

## Recommended Mitigation Steps
pragma solidity 0.7.5;  should be used in all files instead of  pragma solidity ^0.7.5;
The pragma version should be locked on a specific version while considering know bugs for the chosen compiler version.  

