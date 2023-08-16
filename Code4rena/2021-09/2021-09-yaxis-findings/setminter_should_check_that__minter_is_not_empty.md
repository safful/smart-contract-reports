## Tags

- bug
- sponsor acknowledged
- sponsor confirmed
- 0 (Non-critical)

# [setMinter should check that _minter is not empty](https://github.com/code-423n4/2021-09-yaxis-findings/issues/81) 

# Handle

pauliax


# Vulnerability details

## Impact
function setMinter should validate that _minter is not an empty (0x0) address.

## Recommended Mitigation Steps
require(_minter != address(0), "!_minter");

