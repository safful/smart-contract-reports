## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Use `calldata` keyword instead of `memory` keyword in function arguments](https://github.com/code-423n4/2021-11-nested-findings/issues/107) 

# Handle

xYrYuYx


# Vulnerability details

## Impact
`calldata` use less gas than `memory` in function arguments

https://github.com/code-423n4/2021-11-nested/blob/main/contracts/FeeSplitter.sol#L124

## Tools Used
Manual

## Recommended Mitigation Steps
Use `calldata` keyword in function argument instead of `memory`

