## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [OperatorResolver.areAddressesImported doesn't check lengths of argument arrays](https://github.com/code-423n4/2021-11-nested-findings/issues/210) 

# Handle

hyh


# Vulnerability details

## Impact

Array bounds check violation will happen if the function be called with arrays of different lengths.

## Proof of Concept

Loop is performed by names array, while both arrays are accessed:
```
for (uint256 i = 0; i < names.length; i++) {
		if (operators[names[i]] != destinations[i]) {
```
https://github.com/code-423n4/2021-11-nested/blob/main/contracts/OperatorResolver.sol#L27

## Recommended Mitigation Steps

Add a check:
```
require(names.length == destinations.length, "OperatorResolver::areAddressesImported: Input lengths must match");
```

