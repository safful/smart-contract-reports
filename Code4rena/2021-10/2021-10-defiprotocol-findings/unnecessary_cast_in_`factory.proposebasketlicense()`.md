## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Unnecessary cast in `Factory.proposeBasketLicense()`](https://github.com/code-423n4/2021-10-defiprotocol-findings/issues/20) 

# Handle

pants


# Vulnerability details

The function `Factory.proposeBasketLicense()` casts `msg.sender` to type `address`, although it is already a variable of type `address`.

## Impact
A redundent operation is executing.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Remove the cast to `address`.

