## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [uberOwner cannot do all the things an owner can](https://github.com/code-423n4/2021-06-realitycards-findings/issues/156) 

# Handle

cmichel


# Vulnerability details

## Vulnerability Details

The `uberOwner` cannot do the same things the owner can.
They can "only" set the reference contract for the market.

The same ideas apply to `Treasury` and `Factory`'s `uberOwner`.

## Impact

The name is misleading as it sounds like the uber-owner is more powerful than the owner.

## Recommended Mitigation Steps

Uberowner should at least be able to set the owner if not be allowed to call all functions that an `owner` can.
Alternatively, rename the `uberOwner`.

