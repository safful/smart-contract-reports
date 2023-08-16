## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [MAX_TIMEOUT](https://github.com/code-423n4/2021-07-connext-findings/issues/33) 

# Handle

pauliax


# Vulnerability details

## Impact
There is a MIN_TIMEOUT for the expiry but I think you should also introduce a MAX_TIMEOUT to avoid a scenario when, for example, expiry is set far in the future (e.g. 100 years) and one malicious side does not agree to fulfill or cancel the tx so the other side then has to wait and leave the funds locked for 100 years or so.

## Recommended Mitigation Steps
Introduce a reasonable MAX_TIMEOUT.

