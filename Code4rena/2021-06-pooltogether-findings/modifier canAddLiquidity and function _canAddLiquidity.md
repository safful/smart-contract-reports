## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [modifier canAddLiquidity and function _canAddLiquidity](https://github.com/code-423n4/2021-06-pooltogether-findings/issues/19) 

# Handle

pauliax


# Vulnerability details

## Impact
modifier canAddLiquidity calls internal function _canAddLiquidity. This function is not called anywhere else so I do not see a reason why all the logic can't be moved to the modifier to save some gas by reducing the extra call.

## Recommended Mitigation Steps
Remove function _canAddLiquidity, place its logic directly in the canAddLiquidity modifier.

