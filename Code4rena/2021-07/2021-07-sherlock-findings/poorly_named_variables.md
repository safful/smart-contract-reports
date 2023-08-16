## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- disagree with severity

# [Poorly Named variables](https://github.com/code-423n4/2021-07-sherlock-findings/issues/123) 

# Handle

tensors


# Vulnerability details

## Impact
Poorly named variables in Gov.sol

## Proof of Concept
_protocolPremium is a bool while protcolPremium is a mapping to uint. This is confusing a could potentially cause some input errors.

## Recommended Mitigation Steps
Rename variables.

