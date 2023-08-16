## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Redundant Balance Check](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/85) 

# Handle

shenwilly


# Vulnerability details

## Impact
OpenZeppelin ERC20Upgradeable `_burn` already checks for account balance, so another check is unnecessary.


## Proof of Concept
https://github.com/code-423n4/2021-09-defiProtocol/blob/52b74824c42acbcd64248f68c40128fe3a82caf6/contracts/contracts/Basket.sol#L92


## Recommended Mitigation Steps
Remove the require statement

