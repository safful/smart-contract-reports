## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Use unchecked math and cache values](https://github.com/code-423n4/2021-12-nftx-findings/issues/208) 

# Handle

pauliax


# Vulnerability details

## Impact
'unchecked' directive can be used where an underflow/overflow cannot happen, e.g. here:
```solidity
  if (amountEth < msg.value) {
    WETH.withdraw(msg.value-amountEth);
    payable(to).call{value: msg.value-amountEth};
  }
```
Also, to reduce gas usage, ```msg.value-amountEth``` should be cached and not re-calculated several times.

