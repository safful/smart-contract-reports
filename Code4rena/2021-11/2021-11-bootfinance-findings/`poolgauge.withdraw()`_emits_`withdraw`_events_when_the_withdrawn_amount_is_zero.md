## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [`PoolGauge.withdraw()` emits `Withdraw` events when the withdrawn amount is zero](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/49) 

# Handle

pants


# Vulnerability details

The function `PoolGauge.withdraw()` emits `Withdraw` events when the withdrawn amount is zero.

## Impact
There is no reason to emit these `Withdraw` events because nothing has changed in the system. Such events are only going to confuse users.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Emit these events only when the withdrawn amount is not zero.

