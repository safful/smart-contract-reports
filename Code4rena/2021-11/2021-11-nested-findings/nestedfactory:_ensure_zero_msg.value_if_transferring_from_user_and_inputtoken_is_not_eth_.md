## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed

# [NestedFactory: Ensure zero msg.value if transferring from user and inputToken is not ETH ](https://github.com/code-423n4/2021-11-nested-findings/issues/136) 

# Handle

GreyArt


# Vulnerability details

## Impact

A user that mistakenly calls either `create()` or `addToken()` with WETH (or another ERC20) as the input token, but includes native ETH with the function call will have his native ETH permanently locked in the contract.

## Recommended Mitigation Steps

It is best to ensure that `msg.value = 0` in `_transferInputTokens()` for the scenario mentioned above.

```jsx
} else if (address(_inputToken) == ETH) {
	...
} else {
	require(msg.value == 0, "NestedFactory::_transferInputTokens: ETH sent for non-ETH transfer");
  _inputToken.safeTransferFrom(_msgSender(), address(this), _inputTokenAmount);
}
```

