## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Pending governance is not cleared](https://github.com/code-423n4/2021-10-badgerdao-findings/issues/42) 

# Handle

cmichel


# Vulnerability details

The `acceptPendingGovernance` function does not reset `pendingGovernance` to zero.

## Impact
The pending governor can repeatedly accept the governance, emitting an `AcceptPendingGovernance` event each time, bloating listeners for this event with unnecessary data.

## Recommended Mitigation Steps
Validate the parameters.


