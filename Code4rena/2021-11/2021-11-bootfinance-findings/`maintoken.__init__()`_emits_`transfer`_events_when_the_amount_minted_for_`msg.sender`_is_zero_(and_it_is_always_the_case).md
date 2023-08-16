## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [`MainToken.__init__()` emits `Transfer` events when the amount minted for `msg.sender` is zero (and it is always the case)](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/67) 

# Handle

pants


# Vulnerability details

The function `MainToken.__init__()` (the constructor) emits `Transfer` events when the amount minted for `msg.sender` is zero. This is always the case, as the value of the constant `INITIAL_SUPPLY` is zero.

## Impact
There is no reason to emit these `Transfer` events because nothing has changed in the system. Such events are only going to confuse users.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Emit these events only when the amount minted for `msg.sender` is not zero.

