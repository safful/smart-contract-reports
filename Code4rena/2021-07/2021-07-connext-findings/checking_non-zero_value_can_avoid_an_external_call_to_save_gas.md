## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Checking non-zero value can avoid an external call to save gas](https://github.com/code-423n4/2021-07-connext-findings/issues/45) 

# Handle

0xRajeev


# Vulnerability details

## Impact

Checking if toSend > 0 before making the external library call to LibAsset.transferAsset() can save 2600 gas by avoiding the external call in such situations.


## Proof of Concept

https://github.com/code-423n4/2021-07-connext/blob/8e1a7ea396d508ed2ebeba4d1898a748255a48d2/contracts/TransactionManager.sol#L375-L380

https://github.com/code-423n4/2021-07-connext/blob/8e1a7ea396d508ed2ebeba4d1898a748255a48d2/contracts/TransactionManager.sol#L364

## Tools Used

Manual Analysis

## Recommended Mitigation Steps

Add toSend > 0 to predicate on L375 similar to check on L387.

