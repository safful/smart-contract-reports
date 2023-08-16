## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas saving using delete](https://github.com/code-423n4/2021-12-sublime-findings/issues/7) 

# Handle

0x1f8b


# Vulnerability details

## Impact
Gas saving.

## Proof of Concept
In the method updateStrategy and removeStrategy of StrategyRegistry contract, when the contract want to remove a strategy, the old one, it's set to false, instead of use delete, this will remaing the storage space and it has expensive than use delete.

## Tools Used
Manual review

## Recommended Mitigation Steps
Use delete instead of set to `false`

