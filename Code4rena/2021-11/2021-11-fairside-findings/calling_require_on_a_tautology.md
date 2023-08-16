## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Calling require on a tautology](https://github.com/code-423n4/2021-11-fairside-findings/issues/78) 

# Handle

0x0x0x


# Vulnerability details

## Impact
Gas optimization

## Finding
``` 
contracts/token/FSD.sol:174: require(bonded > 0, "FSD::mintHatch: Insufficient Deposit");
```
On L170, we check whether `bonded` is bigger than 5 ETH. After multiplying with `HATCH_CURVE_RATIO`, it is still over 0. Therefore, it is a tautology and not needed.

## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.

## Tools Used

Manual analysis

