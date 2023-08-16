## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [CreditBurned event emitted even on zero tokens burned](https://github.com/code-423n4/2021-06-pooltogether-findings/issues/97) 

# Handle

cmichel


# Vulnerability details

In `PrizePool._updateCreditBalance` the `CreditBurned` event is emitted even if nothing was burned.
Not emitting this event when nothing happened can save gas and also seems better semantically.

