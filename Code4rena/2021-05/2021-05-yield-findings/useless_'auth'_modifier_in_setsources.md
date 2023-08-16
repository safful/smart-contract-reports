## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Useless 'auth' modifier in setSources](https://github.com/code-423n4/2021-05-yield-findings/issues/20) 

# Handle

pauliax


# Vulnerability details

## Impact
function setSources in Oracle contracts does not need 'auth' modifier as it will be checked anyway in function setSource. This does not impact the security, it is just a useless check that can be removed.

## Recommended Mitigation Steps
Remove 'auth' modifer from function setSources.

