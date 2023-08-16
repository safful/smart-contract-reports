## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- resolved

# [MapleTreasury does not emit an event when MapleGlobals address is updated](https://github.com/code-423n4/2021-04-maple-findings/issues/8) 

# Handle

JMukesh


# Vulnerability details

## Impact
Its impact will be limited since we will not able tract the change of address off-chain but on-chain we can which will consume gas

## Proof of Concept
In file MapleTreasury.sol has no event, so it is difficult to track off-chain changes of  Address of new MapleGlobals contract




## Tools Used

slither

## Recommended Mitigation Steps

add event for setting global address

