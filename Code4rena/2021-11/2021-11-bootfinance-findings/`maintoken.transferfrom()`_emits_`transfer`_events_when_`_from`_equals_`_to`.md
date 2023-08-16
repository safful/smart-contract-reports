## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [`MainToken.transferFrom()` emits `Transfer` events when `_from` equals `_to`](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/52) 

# Handle

pants


# Vulnerability details

The function `MainToken.transferFrom()` emits `Transfer` events when `_from` equals `_to`.

## Impact
There is no reason to emit these `Transfer` events because nothing has changed in the system. Such events are only going to confuse users.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Emit these events only when `_from` doesn't equal `_to`.

