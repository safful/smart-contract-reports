## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Join _checkToken function and modifier together](https://github.com/code-423n4/2021-09-yaxis-findings/issues/91) 

# Handle

pauliax


# Vulnerability details

## Impact
function _checkToken can be moved to modifier checkToken as it is a private function that is only used by this modifier. This will reduce the number of extra calls and thus reduce the gas.

## Recommended Mitigation Steps
Consider moving this function inside the modifier to reduce gas usage.

