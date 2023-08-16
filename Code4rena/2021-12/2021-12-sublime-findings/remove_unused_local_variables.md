## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Remove unused local variables](https://github.com/code-423n4/2021-12-sublime-findings/issues/119) 

# Handle

pmerkleplant


# Vulnerability details

## Impact

The local variables `_receivedToken` in the functions `SavingsAccount.withdraw` and
`SavingsAccount.withdrawFrom` are unused.

Removing them would save gas.

## Tools used

slither

