## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [addValidator(): Validator's commission rate should be checked to not exceed divider](https://github.com/code-423n4/2021-10-covalent-findings/issues/20) 

# Handle

hickuphh3


# Vulnerability details

### Impact

The check `require(amount < divider, "Rate must be less than 100%");` exists in `setValidatorComissionRate()` but not in `addValidator()`.

### Recommended Mitigation Steps

Add the check in `addValidator()` as well.

