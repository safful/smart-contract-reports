## Tags

- bug
- 0 (Non-critical)
- disagree with severity
- sponsor confirmed

# [Floating pragma](https://github.com/code-423n4/2021-12-maple-findings/issues/23) 

# Handle

saian


# Vulnerability details

## Impact

Contracts should be deployed with the same version of compilers with which it was tested, 
Using a unlocked pragma might result in contract being deployed with a version it was not tested with, and might result in bugs and unwanted behaviour.


## Proof of Concept

Contracts in below repositories :
    maple-labs/debt-locker
    maple-labs/erc20-helper
    maple-labs/loan
    maple-labs/maple-proxy-factory
    maple-labs/proxy-factory


## Tools Used

Manual Analysis

## Recommended Mitigation Steps

Lock the pragma version, it is advised not to use unlocked pragma in production.

