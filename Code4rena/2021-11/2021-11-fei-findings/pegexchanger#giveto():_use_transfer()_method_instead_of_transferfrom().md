## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [PegExchanger#giveTo(): Use transfer() method instead of transferFrom()](https://github.com/code-423n4/2021-11-fei-findings/issues/104) 

# Handle

hickuphh3


# Vulnerability details

## Impact

Looking at the TRIBE token implementation, it would be cheaper to call the `transfer()` method as opposed to the the `transferFrom()` method since the latter contains additional logic (Eg. additional SLOAD to fetch the allowance).

## Recommended Mitigation Steps

```jsx
function giveTo(address target, uint256 amount) internal {
  bool check = token1.transfer(target, amount);
	require(check, "erc20 transfer failed");
}
```

