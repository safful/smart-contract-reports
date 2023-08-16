## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed

# [Incentive should check that it hasn't started yet](https://github.com/code-423n4/2021-09-sushitrident-2-findings/issues/42) 

# Handle

cmichel


# Vulnerability details

The `ConcentratedLiquidityPoolManager.addIncentive` function can add an incentive that already has a non-zero `incentive.secondsClaimed`.

## Impact
Rewards will be wrong.

## Recommended Mitigation Steps
Add a check: `require(incentive.secondsClaimed == 0, "!secondsClaimed")`.


