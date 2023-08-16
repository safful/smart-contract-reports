## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [FSDVesting: Redundant _start input parameter in initiateVesting()](https://github.com/code-423n4/2021-11-fairside-findings/issues/33) 

# Handle

hickuphh3


# Vulnerability details

## Impact

Tracing the function calls, the `_start` parameter in `initiateVesting()` will always be `block.timestamp`. Hence, this input parameter can be removed from the function.

## Recommended Mitigation Steps

```jsx
// TODO: Modify relevant function calls
function initiateVesting(
  address _beneficiary,
  uint256 _amount
) external onlyFactory {
	require(
    start == 0,
    "FSDVesting::initiateVesting: Vesting is already initialized"
  );
	beneficiary = _beneficiary;
	start = block.timestamp;
	amount = _amount;
}
```

