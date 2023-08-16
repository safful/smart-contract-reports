## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [takeOutRewardTokens(): Optimise epochs calculation and comparison ](https://github.com/code-423n4/2021-10-covalent-findings/issues/26) 

# Handle

hickuphh3


# Vulnerability details

## Impact

The following lines in `takeOutRewardTokens()` are only needed in the case where `endEpoch != 0`.

```jsx
uint128 currentEpoch = uint128(block.number);
uint128 epochs = amount / allocatedTokensPerEpoch;
```

Hence, they can be shifted inside the "if" block.

Furthermore, a double calculation of `endEpoch - epochs` can be avoided by saving the result into a new variable `newEpoch`.

## Recommended Mitigation Steps

```jsx
if (endEpoch != 0) {
  uint128 newEpoch = endEpoch - (amount / allocatedTokensPerEpoch);
  require(newEpoch  > uint128(block.number), "Cannot takeout rewards from past");
  endEpoch = newEpoch;
}
```

