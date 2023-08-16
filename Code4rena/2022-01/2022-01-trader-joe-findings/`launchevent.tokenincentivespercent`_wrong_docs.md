## Tags

- bug
- 1 (Low Risk)
- disagree with severity
- sponsor confirmed

# [`LaunchEvent.tokenIncentivesPercent` wrong docs](https://github.com/code-423n4/2022-01-trader-joe-findings/issues/208) 

# Handle

cmichel


# Vulnerability details

`LaunchEvent.tokenIncentivesPercent`: The math in the comment is wrong: `/// then 105 000 * 1e18 / (1e18 + 5e16) = 5 000 tokens are used for incentives`. It should be `105 000 * 5e16 / (1e18 + 5e16) = 5 000 tokens are used for incentives`

