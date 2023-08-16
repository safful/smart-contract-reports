## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Unnecessary cast in `Basket.onlyPublisher()`](https://github.com/code-423n4/2021-10-defiprotocol-findings/issues/21) 

# Handle

pants


# Vulnerability details

The modifier `Basket.onlyPublisher()` casts `publisher` to type `address`, although it is already a variable of type `address`.

## Impact
A redundent operation is executing.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Remove the cast to `address`.

