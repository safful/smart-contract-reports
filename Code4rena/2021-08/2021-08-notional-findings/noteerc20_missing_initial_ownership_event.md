## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [NoteERC20 missing initial ownership event](https://github.com/code-423n4/2021-08-notional-findings/issues/74) 

# Handle

cmichel


# Vulnerability details

## Vulnerability Details
The `NoteERC20.initialize` function does not emit an initial `OwnershipTransferred` event.

## Recommended Mitigation Steps
In `initialize`, emit `OwnershipTransferred(address(0), owner_)`.


