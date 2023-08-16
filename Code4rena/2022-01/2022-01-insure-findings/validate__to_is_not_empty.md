## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [Validate _to is not empty](https://github.com/code-423n4/2022-01-insure-findings/issues/314) 

# Handle

pauliax


# Vulnerability details

## Impact
_withdrawAttribution should validate that _to is not an empty address 0x0 to prevent accidental burns. Similarly, transferValue _destination param and withdrawValue _to param should also be checked against an empty address unless this is the intended functionality in some cases.

## Recommended Mitigation Steps
require _to != address(0)

