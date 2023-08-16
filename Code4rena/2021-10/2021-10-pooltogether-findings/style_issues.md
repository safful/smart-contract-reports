## Tags

- bug
- sponsor confirmed
- 0 (Non-critical)
- resolved

# [Style issues](https://github.com/code-423n4/2021-10-pooltogether-findings/issues/52) 

# Handle

pauliax


# Vulnerability details

## Impact
Style issues that you may want to apply or reject, no impact on security. Grouping them together as one submission to reduce waste. Consider fixing or ignoring them, up to you.

* function controllerBurnFrom could also skip _approve decrease if the current approval is uint max (unlimited).

* hardcoded number 1000 in PrizeSplit could be extracted to a constant variable to improve readability and maintainability.

