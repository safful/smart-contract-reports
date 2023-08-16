## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed

# [createBasket re-entrancy](https://github.com/code-423n4/2021-10-defiprotocol-findings/issues/85) 

# Handle

pauliax


# Vulnerability details

## Impact
function createBasket in Factory should also be nonReentrant as it interacts with various tokens inside the loop and these tokens may contain callback hooks.

## Recommended Mitigation Steps
Add nonReentrant modifier to the declaration of createBasket.

