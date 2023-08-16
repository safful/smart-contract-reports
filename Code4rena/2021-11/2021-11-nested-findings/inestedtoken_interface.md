## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [INestedToken interface](https://github.com/code-423n4/2021-11-nested-findings/issues/206) 

# Handle

pauliax


# Vulnerability details

## Impact
INestedToken is declared as an abstract contract, yet it contains no function bodies and is located under the interfaces directory, so I think it should be declared as an interface.

## Recommended Mitigation Steps
Consider making INestedToken an interface.

