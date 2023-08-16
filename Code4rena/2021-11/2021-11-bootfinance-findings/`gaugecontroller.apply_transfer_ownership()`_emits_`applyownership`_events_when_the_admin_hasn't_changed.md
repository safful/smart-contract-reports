## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [`GaugeController.apply_transfer_ownership()` emits `ApplyOwnership` events when the admin hasn't changed](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/80) 

# Handle

pants


# Vulnerability details

The function `GaugeController.apply_transfer_ownership()` emits `ApplyOwnership` events when the admin hasn't changed and left as it was before that transaction.

## Impact
There is no reason to emit these `ApplyOwnership` events because nothing has changed in the system. Such events are only going to confuse users.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Emit these events only when the new admin is different than the old one.

