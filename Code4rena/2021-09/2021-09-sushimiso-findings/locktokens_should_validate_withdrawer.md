## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [lockTokens should validate withdrawer](https://github.com/code-423n4/2021-09-sushimiso-findings/issues/92) 

# Handle

pauliax


# Vulnerability details

## Impact
function lockTokens in contract TokenVault should check that _withdrawer is not empty (0x0) to prevent accidentally locked forever (burned) tokens.

## Recommended Mitigation Steps
require(_withdrawer != address(0));

