## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [setBigFishThreshold above 100%](https://github.com/code-423n4/2021-06-gro-findings/issues/80) 

# Handle

pauliax


# Vulnerability details

## Impact
function setBigFishThreshold should require that _percent is not above PERCENTAGE_DECIMAL_FACTOR if it is not intended to have it over 100%.

## Recommended Mitigation Steps
require _percent <= PERCENTAGE_DECIMAL_FACTOR

