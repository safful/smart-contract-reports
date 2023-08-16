## Tags

- bug
- duplicate
- 0 (Non-critical)
- sponsor confirmed
- resolved

# [Comment indicates that FundsWithdrawn event should be emitted only when _withdrawableDividend > 0](https://github.com/code-423n4/2021-04-maple-findings/issues/96) 

# Handle

paulius.eth


# Vulnerability details

## Vulnerability details

A comment says: "It emits a `FundsWithdrawn` event if the amount of withdrawn ether is greater than 0." However, actually, this event is always emitted (no check against 0).


## Recommended Mitigation Steps
Either emit this event if _withdrawableDividend > 0 or remove the comment.



