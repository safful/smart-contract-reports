## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [double reading from memory inside a for loop.](https://github.com/code-423n4/2021-10-union-findings/issues/103) 

# Handle

pants


# Vulnerability details

See for example the AssetsManager.rebalance function. There you read the value moneyMarkets[i] twice at the same iteration instead of caching it. This happens in the same file in many other places, deposit, withdraw and more.
Inside a loop caching is very important. 

