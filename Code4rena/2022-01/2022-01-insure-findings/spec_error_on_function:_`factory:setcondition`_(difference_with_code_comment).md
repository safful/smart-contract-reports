## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [Spec error on function: `Factory:setCondition` (difference with code comment)](https://github.com/code-423n4/2022-01-insure-findings/issues/297) 

# Handle

Dravee


# Vulnerability details

The spec says the function should be called `approveCondition()` instead of `setCondition`: https://insuredao.gitbook.io/developers/market/factory#approvecondition

While this might still be understood nonetheless as `setCondition` is also mentioned, the spec says that the parameter `_slot` is the `index of the reference array`, whereas the code comment says it's the `index within condition array`: https://github.com/code-423n4/2022-01-insure/blob/main/contracts/Factory.sol#L133

## Tools Used
VS Code

## Recommended Mitigation Steps
My guess is that the spec should be corrected

