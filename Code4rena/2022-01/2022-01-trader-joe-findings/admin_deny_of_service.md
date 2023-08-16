## Tags

- bug
- 1 (Low Risk)
- disagree with severity
- sponsor confirmed

# [Admin Deny of Service](https://github.com/code-423n4/2022-01-trader-joe-findings/issues/25) 

# Handle

0x1f8b


# Vulnerability details

## Impact
Owner can Denial of service.

## Proof of Concept
In the contract `RocketJoeStaking` there are two ways to set `rJoePerSec`, one in the `initialize` and the second one in `updateEmissionRate`, in both of them there are no checks of the received value, so it's possible to use a high value and deny the service in line `updatePool:168`.

## Tools Used
Manual review

## Recommended Mitigation Steps
Change the type to uint128 for `rJoePerSec`.

