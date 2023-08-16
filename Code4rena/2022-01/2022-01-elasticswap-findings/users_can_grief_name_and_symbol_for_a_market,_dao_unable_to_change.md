## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Users can grief name and symbol for a market, DAO unable to change](https://github.com/code-423n4/2022-01-elasticswap-findings/issues/110) 

# Handle

camden


# Vulnerability details

## Impact
https://github.com/ElasticSwap/elasticswap/blob/a90bb67e2817d892b517da6c1ba6fae5303e9867/src/contracts/ExchangeFactory.sol#L38

A user could create an exchange with a name and symbol that is misleading or allows phishing into an exchange created with an unexpected token.

## Recommended Mitigation Steps
Allow the ExchangeFactory to change the name and symbol of an exchange.

