## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [RewardReinvestor - safeTransfer used unnecessarily on Malt token](https://github.com/code-423n4/2021-11-malt-findings/issues/339) 

# Handle

ScopeLift


# Vulnerability details

## Impact

Inside `provideReinvest` and `_bondAccount` gas can be saved by using the standard transfer method on the Malt token, since we know its implementation is correct and will return true/false.

## Proof of Concept

N/A

## Tools Used

N/A

## Recommended Mitigation Steps

Replace `malt.safeTransfer(address(dexHandler), balance);` with something like:

```solidity
require(malt.transfer(address(dexHandler), balance), 'malt transfer failed');
```


