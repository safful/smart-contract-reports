## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed
- Timelock

# [TimeLock cannot schedule the same calls multiple times](https://github.com/code-423n4/2021-08-yield-findings/issues/27) 

# Handle

cmichel


# Vulnerability details

The `TimeLock.schedule` function reverts if the same `targets` and `data` fields are used as the `txHash` will be the same.
This means one cannot schedule the same transactions multiple times.

## Impact
Imagine the delay is set to 30 days, but a contractor needs to be paid every 2 weeks.
One needs to wait 30 days before scheduling the second payment to them.

## Recommended Mitigation Steps
Also include `eta` in the hash. (Compound's Timelock does it as well.) This way the same transaction data can be used by specifying a different `eta`.


