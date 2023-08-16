## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Can set values to more than 100%](https://github.com/code-423n4/2021-06-tracer-findings/issues/102) 

# Handle

cmichel


# Vulnerability details

There are several setter functions that do not check if the amount is less than 100%.

- `TracerPerpetualSwaps`: `setFeeRate`, `setDeleveragingCliff`, `setInsurancePoolSwitchStage`
- `Insurance`: `setFeeRate`, `setDeleveragingCliff`, `setInsurancePoolSwitchStage`

## Impact
Setting values to more than 100% might lead to unintended functionality.

## Recommended Mitigation Steps
Ensure that the parameters are less than 100%.

