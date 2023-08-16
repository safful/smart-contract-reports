## Tags

- bug
- 0 (Non-critical)
- resolved
- sponsor confirmed

# [Context and msg.sender](https://github.com/code-423n4/2021-11-yaxis-findings/issues/105) 

# Handle

pauliax


# Vulnerability details

## Impact
Contract Transmuter inherits a functionality of the Context contract of OpenZeppelin:
```solidity
  contract Transmuter is Context
```
https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Context.sol
Context is designed to be used with Ethereum Gas Station Network
(GSN), thus it encourages to use _msgSender() instead of msg.sender.

## Recommended Mitigation Steps
Consider replacing msg.sender with _msgSender() or getting rid of Context inheritance to save some gas if you don't actually need it.


