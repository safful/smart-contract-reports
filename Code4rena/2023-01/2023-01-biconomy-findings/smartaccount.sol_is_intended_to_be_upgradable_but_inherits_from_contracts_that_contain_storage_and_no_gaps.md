## Tags

- bug
- 2 (Med Risk)
- judge review requested
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-07

# [SmartAccount.sol is intended to be upgradable but inherits from contracts that contain storage and no gaps](https://github.com/code-423n4/2023-01-biconomy-findings/issues/261) 

# Lines of code

https://github.com/code-423n4/2023-01-biconomy/blob/53c8c3823175aeb26dee5529eeefa81240a406ba/scw-contracts/contracts/smart-contract-wallet/base/ModuleManager.sol#L18


# Vulnerability details

## Impact

SmartAccount.sol inherits from contracts that are not stateless and don't contain storage gaps which can be dangerous when upgrading

## Proof of Concept

When creating upgradable contracts that inherit from other contracts is important that there are storage gap in case storage variable are added to inherited contracts. If an inherited contract is a stateless contract (i.e. it doesn't have any storage) then it is acceptable to omit a storage gap, since these function similar to libraries and aren't intended to add any storage. The issue is that SmartAccount.sol inherits from contracts that contain storage that don't contain any gaps such as ModuleManager.sol. These contracts can pose a significant risk when updating a contract because they can shift the storage slots of all inherited contracts.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Add storage gaps to all inherited contracts that contain storage variables