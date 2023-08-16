## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [`nonReentrant` modifier should be used before any other modifier](https://github.com/code-423n4/2021-10-defiprotocol-findings/issues/45) 

# Handle

pants


# Vulnerability details

The function `Basket.auctionBurn()` uses the `onlyAuction` and `nonReentrant` modifier, with this order.

## Impact
The `nonReentrant` modifier doesn't protect agains reentrancy during the execution of the first modifier. Practically, there cannot be any reentrancy there when considering the current implementation of `onlyAuction`, but it is still a best practice recommendation for safe programming.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Use the `nonReentrant` modifier before any other modifier.

