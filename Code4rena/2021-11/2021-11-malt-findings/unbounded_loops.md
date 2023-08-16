## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Unbounded loops](https://github.com/code-423n4/2021-11-malt-findings/issues/358) 

# Handle

pauliax


# Vulnerability details

## Impact
There are several loops in the contract which can eventually grow so large as to make future operations of the contract cost too much gas to fit in a block, e.g.:
```solidity
  for (uint256 i = replenishingIndex; i < auctionIds.length; i = i + 1) // function outstandingArbTokens()
  while (true) // function allocateArbRewards
```

## Recommended Mitigation Steps
Consider introducing a reasonable upper limit based on block gas limits. Also, you can consider using EnumerableSet (https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol) where possible, e.g. 'buyers' or 'verifierList'.

