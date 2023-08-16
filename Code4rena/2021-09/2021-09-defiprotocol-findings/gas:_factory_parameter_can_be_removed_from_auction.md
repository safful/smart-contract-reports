## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: Factory parameter can be removed from Auction](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/225) 

# Handle

cmichel


# Vulnerability details

The `Auction.initialize` function accepts a `factory_` parameter.
However, as this contract is always initialized directly from the factory, it can just use `msg.sender`.

## Recommended Mitigation Steps
Removing the additional `factory_` parameter and using `msg.sender` instead will save gas.
This is already done for the other `Basket` contract.


