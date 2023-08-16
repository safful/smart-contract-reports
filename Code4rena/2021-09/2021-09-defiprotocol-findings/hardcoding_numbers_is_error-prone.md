## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Hardcoding numbers is error-prone](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/203) 

# Handle

pauliax


# Vulnerability details

## Impact
Hardcoding numbers that depend on other variables is error-prone, e.g.
    require(newOwnerSplit <= 2e17); // 20%
You must not forget to update this if you decide to change the BASE value.

## Recommended Mitigation Steps
 Better define a separate constant that directly depends on the BASE, e.g.:
    uint256 private constant MAX_OWNER_SPLIT = BASE / 5; // 20%
    require(newOwnerSplit <= MAX_OWNER_SPLIT);

