## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed
- resolved

# [Gas: `PrizePool.captureAwardBalance` computation can be simplified](https://github.com/code-423n4/2021-10-pooltogether-findings/issues/29) 

# Handle

cmichel


# Vulnerability details

The `PrizePool.captureAwardBalance` function always sets `_currentAwardBalance = currentAwardBalance` where `currentAwardBalance = currentAwardBalance + unaccountedPrizeBalance = currentAwardBalance + totalInterest - currentAwardBalance = totalInterest`.

Save a checked math addition by just setting `_currentAwardBalance = totalInterest` immediately.

