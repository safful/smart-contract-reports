## Tags

- bug
- 0 (Non-critical)
- resolved
- sponsor confirmed

# [`ConvexStakingWrapper.sol`: `AccessControl` capabilities aren't used](https://github.com/code-423n4/2022-01-yield-findings/issues/75) 

# Handle

Dravee


# Vulnerability details

## Impact
AccessControl capabilities aren't used.

## Proof of Concept
In `ConvexStakingWrapper.sol`, `AccessControl` seem superfluous:
```
7: import "@yield-protocol/utils-v2/contracts/access/AccessControl.sol";
...
15: contract ConvexStakingWrapper is ERC20, AccessControl {
```
In the original `ConvexStakingWrapper.sol`, this `AccessControl` isn't inherited. 

In this contract, I believe role-based capabilities were thought of, but were forgotten or abandonned.

## Tools Used
VS Code

## Recommended Mitigation Steps
Either use the capabilities from `AccessControl`, or delete the import + the inheritance to save gas.

