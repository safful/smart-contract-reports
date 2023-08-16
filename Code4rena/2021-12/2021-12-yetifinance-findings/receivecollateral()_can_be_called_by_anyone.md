## Tags

- bug
- 3 (High Risk)
- sponsor confirmed

# [receiveCollateral() can be called by anyone](https://github.com/code-423n4/2021-12-yetifinance-findings/issues/74) 

# Handle

jayjonah8


# Vulnerability details

## Impact
In StabilityPool.sol, the receiveCollateral() function should be called by ActivePool per comments,  but anyone can call it passing in _tokens and _amounts args to update stability pool balances. 

## Proof of Concept
https://github.com/code-423n4/2021-12-yetifinance/blob/main/packages/contracts/contracts/StabilityPool.sol#L1143

## Tools Used
Manual code review 

## Recommended Mitigation Steps
Allow only the ActivePool to call the receiveCollateral() function:
require(msg.sender = address(active pool address), "Can only be called by ActivePool")

