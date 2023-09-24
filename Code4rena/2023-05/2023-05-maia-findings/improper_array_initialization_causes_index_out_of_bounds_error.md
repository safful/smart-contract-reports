## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-44

# [Improper array initialization causes index out of bounds error](https://github.com/code-423n4/2023-05-maia-findings/issues/25) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/ccc9a39240dbd8eab22299737370996b2b833efd/src/ulysses-amm/factories/UlyssesFactory.sol#L93


# Vulnerability details

## Impact
In `createPools` of UlyssesFactory.sol, the return parameter `poolIds` is used to store new pool ids after creation. However, it has not been initialized. This causes an index out of bounds error when `createPools` is called.

## Proof of Concept
Any test that calls `ulyssesFactory.createPools(...);` will cause index out of bounds

## Tools Used
Manual review

## Recommended Mitigation Steps
Consider adding this line:
```
poolIds = new uint256[](length);
```


## Assessed type

Invalid Validation