## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [NestedRecords: Unnecessary variable in the Holding struct](https://github.com/code-423n4/2021-11-nested-findings/issues/121) 

# Handle

GreyArt


# Vulnerability details

## Impact

It is unnecessary to store the `token` variable in the `Holding` struct because the token is used as the key to access the `Holding` struct.

## Recommended Mitigation Steps

Remove the `token` variable in the `Holding` struct.

```jsx
/// @dev Info about assets stored in reserves
struct Holding {
  uint256 amount;
  bool isActive;
}
```

