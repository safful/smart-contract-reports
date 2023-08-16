## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Cache repeated calculations](https://github.com/code-423n4/2021-12-yetifinance-findings/issues/294) 

# Handle

pauliax


# Vulnerability details

## Impact
In function _transfer, shares.to128(); can be cached to skip the same calculation again:
```solidity
  users[from].balance = fromUser.balance - shares.to128();
  users[to].balance = toUser.balance + shares.to128();
```
Same here, the result can be extracted to a constant as it never changes:
```solidity
  (DECIMAL_PRECISION / 2)
```

