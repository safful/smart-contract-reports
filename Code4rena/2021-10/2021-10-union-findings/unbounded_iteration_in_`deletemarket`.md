## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Unbounded iteration in `deleteMarket`](https://github.com/code-423n4/2021-10-union-findings/issues/65) 

# Handle

cmichel


# Vulnerability details

The `MarketRegistry.deleteMarket` iterates over all `uTokenList` elements.

## Impact
The transactions can fail if the arrays get too big and the transaction would consume more gas than the block limit.
This will then result in a denial of service for the desired functionality and break core functionality.

## Recommended Mitigation Steps
Keep the array small or use an [EnumerableSet])(https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/EnumerableSet.sol) that can delete in constant time.

