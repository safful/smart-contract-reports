## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Unnecessary `SLOAD`s in `Factory`](https://github.com/code-423n4/2021-10-defiprotocol-findings/issues/35) 

# Handle

pants


# Vulnerability details

The functions `Factory.getProposalWeights()` and `Factory.createBasket()` read values from storage multiple times instead of caching them in local variables:
- `Factory.getProposalWeights()` reads `_proposals[id]` twice.
- `Factory.createBasket()` reads `_proposals[idNumber]` twice.

## Impact
Storage reads are much more expensive than reading local variables.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Read these values from storage once, cache them in local variables and then read them again from the local variables.

