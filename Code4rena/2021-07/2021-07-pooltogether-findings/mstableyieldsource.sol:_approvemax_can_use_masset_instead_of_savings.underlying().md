## Tags

- bug
- G (Gas Optimization)
- mStableYieldSource
- sponsor confirmed

# [MStableYieldSource.sol: approveMax can use mAsset instead of savings.underlying()](https://github.com/code-423n4/2021-07-pooltogether-findings/issues/19) 

# Handle

hickuphh3


# Vulnerability details

### Impact

The immutable `mAsset` is assigned to the immutable `savings` contract. Hence, we can avoid an external function call to the savings contract in the `approveMax` function by replacing it with `mAsset`. 

### Recommended Mitigation Steps

```jsx
function approveMax() public {
	mAsset.safeApprove(address(savings), type(uint256).max);

	emit ApprovedMax(msg.sender);
}
```

