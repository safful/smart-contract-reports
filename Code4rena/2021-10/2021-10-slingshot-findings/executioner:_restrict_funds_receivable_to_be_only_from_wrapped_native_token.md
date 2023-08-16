## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Executioner: Restrict funds receivable to be only from wrapped native token](https://github.com/code-423n4/2021-10-slingshot-findings/issues/40) 

# Handle

hickuphh3


# Vulnerability details

## Impact

Native fund transfers into the executioner contract are only expected from the wrapped token contract. Hence, it would be good to restrict incoming fund transfers to prevent accidental native fund transfers from other sources.

## Recommended Mitigation Steps

Modify the `receive()` function to only accept transfers from the wrapped token contract.

```jsx
receive() external payable {
  require(msg.sender == address(wrappedNativeToken), 'only wrapped native token');
}
```

