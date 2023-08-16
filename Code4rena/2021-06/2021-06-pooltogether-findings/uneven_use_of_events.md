## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Uneven use of events](https://github.com/code-423n4/2021-06-pooltogether-findings/issues/78) 

# Handle

JMukesh


# Vulnerability details

## Impact
To track off-chain data it is necessary to use events

## Proof of Concept

In  ATokenYieldSource.sol, IdleYieldSource.sol, yearnV2yieldsource  : events are emmitted in supplyTokenTo(), redeemToken() sponsor(), but not in  BadgerYieldsource.sol and shushiyieldsource.sol

 

## Tools Used

Manual analysis

## Recommended Mitigation Steps
use events 

