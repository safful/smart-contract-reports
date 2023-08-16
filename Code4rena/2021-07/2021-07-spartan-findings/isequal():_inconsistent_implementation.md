## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [isEqual(): Inconsistent Implementation](https://github.com/code-423n4/2021-07-spartan-findings/issues/54) 

# Handle

hickuphh3


# Vulnerability details

```jsx
function isEqual(bytes memory part1, bytes memory part2) external pure returns(bool equal){
  if(sha256(part1) == sha256(part2)){
      return true;
  }
}
```

Both implementations can be simplified and made consistent to be

```jsx
function isEqual(bytes memory part1, bytes memory part2) external pure returns(bool){
  return(sha256(part1) == sha256(part2));
}
```

