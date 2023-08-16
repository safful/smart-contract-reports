## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Unnecessary type conversions](https://github.com/code-423n4/2021-06-tracer-findings/issues/118) 

# Handle

tensors


# Vulnerability details

## Impact
Superfluous type conversions.

## Proof of Concept
https://github.com/code-423n4/2021-06-tracer/blob/main/src/contracts/lib/LibBalances.sol#L229-L230
The type conversion here is not necessary.

## Recommended Mitigation Steps
Remove the type conversion.

