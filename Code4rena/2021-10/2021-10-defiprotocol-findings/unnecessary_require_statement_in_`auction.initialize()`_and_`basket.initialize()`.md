## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Unnecessary require statement in `Auction.initialize()` and `Basket.initialize()`](https://github.com/code-423n4/2021-10-defiprotocol-findings/issues/29) 

# Handle

pants


# Vulnerability details

The functions `Auction.initialize()` and `Basket.initialize()` look like this:
```
require(address(factory) == address(0));
require(!initialized);
// ...
factory = ...;
// ...
initialized = true;

```

The second require statement is enough to make sure that these functions can only be called once. The first require statement is redundent.

## Impact
A redundent operation is executing.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Remove the first require statement in these functions.

