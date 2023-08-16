## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [No check that _factory and _weth are different addresses in constructor ](https://github.com/code-423n4/2022-01-timeswap-findings/issues/45) 

# Handle

jayjonah8


# Vulnerability details

## Impact
In TimeswapConvenience.sol the constructor takes in 2 addresses for _factory and _weth and sets them in storage without checking that they are unique which can introduce possible costly errors during deployment. 

## Proof of Concept
https://github.com/code-423n4/2022-01-timeswap/blob/main/Timeswap/Timeswap-V1-Convenience/contracts/TimeswapConvenience.sol#L62

## Tools Used
Manuel code review 

## Recommended Mitigation Steps
Add to TimeswapConvenience.sol constructor:   require(_factory != _weth, "Duplicate address")

