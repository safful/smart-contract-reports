## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [emit NewIBRatio in function initialize](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/205) 

# Handle

pauliax


# Vulnerability details

## Impact
I think function initialize should also emit NewIBRatio event as it sets the initial value:
 ibRatio = BASE;

## Recommended Mitigation Steps
emit NewIBRatio(ibRatio) in function initialize.

