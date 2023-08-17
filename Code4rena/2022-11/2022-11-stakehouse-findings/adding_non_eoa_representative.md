## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-05

# [Adding non EOA representative](https://github.com/code-423n4/2022-11-stakehouse-findings/issues/93) 

# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L308
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L289


# Vulnerability details

## Impact
It is not allowed to add non-EOA representative to the smart wallet.
But, this limitation can be bypassed by rotating representatives.

## Proof of Concept

During registering a node runner to LSD by creating a new smart wallet, it is checked that the `_eoaRepresentative` is an EOA or not.
```
require(!Address.isContract(_eoaRepresentative), "Only EOA representative permitted");
```
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L426

But this check is missing during rotating EOA representative in two functions `rotateEOARepresentative` and `rotateEOARepresentativeOfNodeRunner`.

https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L289
https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/LiquidStakingManager.sol#L308

In other words `_newRepresentative` can be a contract in these two functions without being prevented. So, this can bypass the check during registering a node runner to LSD.

## Tools Used

## Recommended Mitigation Steps
The following line should be added to functions `rotateEOARepresentative` and `rotateEOARepresentativeOfNodeRunner`:
```
require(!Address.isContract(_newRepresentative), "Only EOA representative permitted");
```