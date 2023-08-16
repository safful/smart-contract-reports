## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Style issues](https://github.com/code-423n4/2021-09-sushitrident-2-findings/issues/74) 

# Handle

pauliax


# Vulnerability details

## Impact
Style issues that you may want to apply or reject, no impact on security. Grouping them together as one submission to reduce waste. Consider fixing or ignoring them, up to you.

* I think the error message here should be "NOT_EXPIRED":
    require(incentive.expiry < block.timestamp, "EXPIRED");

* There are hardcoded magic numbers, e.g.: 5 weeks or 128. It would make code more readable and maintanable if you extract such numbers as constants, e.g.:
  uint public constant EXPIRY_BUFFER = 5 weeks;
  require(incentive.endTime + EXPIRY_BUFFER < incentive.expiry, "END_PAST_BUFFER");


