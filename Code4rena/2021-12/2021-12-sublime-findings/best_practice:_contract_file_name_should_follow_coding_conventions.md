## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Best Practice: Contract file name should follow coding conventions](https://github.com/code-423n4/2021-12-sublime-findings/issues/105) 

# Handle

WatchPug


# Vulnerability details

Having a consistent naming style in the project leads to fewer errors.

https://github.com/code-423n4/2021-12-sublime/blob/9df1b7c4247f8631647c7627a8da9bdc16db8b11/contracts/Proxy.sol#L6-L6

```solidity=6
contract SublimeProxy is TransparentUpgradeableProxy {
```

The filename `Proxy.sol` should be `SublimeProxy.sol`.

