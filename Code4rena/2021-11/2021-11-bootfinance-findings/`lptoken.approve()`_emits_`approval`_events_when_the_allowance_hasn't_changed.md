## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [`LPToken.approve()` emits `Approval` events when the allowance hasn't changed](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/73) 

# Handle

pants


# Vulnerability details

The function `LPToken.approve()` emits `Approval` events when the allowance hasn't changed and left as it was before that transaction.

## Impact
There is no reason to emit these `Approval` events because nothing has changed in the system. Such events are only going to confuse users.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Emit these events only when the new allowance is different than the old one.

