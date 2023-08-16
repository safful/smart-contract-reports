## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- resolved

# [contract RCTreasury does not use nfthub and setNftHubAddress](https://github.com/code-423n4/2021-06-realitycards-findings/issues/35) 

# Handle

pauliax


# Vulnerability details

## Impact
contract RCTreasury has an unused storage variable nfthub and setNftHubAddress function. This variable was moved to the Factory contract so it is useless here.

## Recommended Mitigation Steps
Remove nfthub variable and function setNftHubAddress.

