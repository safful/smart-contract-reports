## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [`MainToken.transfer()` and `MainToken.transferFrom()` emit `Transfer` events when the transferred amount is zero](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/51) 

# Handle

pants


# Vulnerability details

The functions `MainToken.transfer()` and `MainToken.transferFrom()` emit `Transfer` events when the transferred amount is zero.

## Impact
There is no reason to emit these `Transfer` events because nothing has changed in the system. Such events are only going to confuse users.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Emit these events only when the transferred amount is not zero.

