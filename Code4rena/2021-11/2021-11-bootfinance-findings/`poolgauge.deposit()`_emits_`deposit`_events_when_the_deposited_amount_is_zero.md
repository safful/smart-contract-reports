## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [`PoolGauge.deposit()` emits `Deposit` events when the deposited amount is zero](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/48) 

# Handle

pants


# Vulnerability details

The function `PoolGauge.deposit()` emits `Deposit` events when the deposited amount is zero.

## Impact
There is no reason to emit these `Deposit` events because nothing has changed in the system. Such events are only going to confuse users.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Emit these events only when the deposited amount is not zero.

