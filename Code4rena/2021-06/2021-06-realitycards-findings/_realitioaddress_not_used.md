## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- resolved

# [_realitioAddress not used](https://github.com/code-423n4/2021-06-realitycards-findings/issues/7) 

# Handle

gpersoon


# Vulnerability details

## Impact
The variable _realitioAddress of RCMarket.sol isn't used. The variable realitio seems to used instead.
Two variables with the same purpose is confusing.

## Proof of Concept
https://github.com/code-423n4/2021-06-realitycards/blob/main/contracts/RCMarket.sol#L121
    IRealitio public realitio;
    address public _realitioAddress;

## Tools Used

## Recommended Mitigation Steps
Remove     address public _realitioAddress;

