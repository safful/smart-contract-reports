## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [using empty String which is already default 0x](https://github.com/code-423n4/2022-01-livepeer-findings/issues/30) 

# Handle

Tomio


# Vulnerability details

## Impact
expensive gas

## Proof of Concept
https://github.com/livepeer/arbitrum-lpt-bridge/blob/main/contracts/L1/gateway/L1LPTGateway.sol#L227

## Tools Used
Remix

## Recommended Mitigation Steps
change to `bytes memory emptyBytes;`

