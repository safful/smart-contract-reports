## Tags

- bug
- disagree with severity
- G (Gas Optimization)
- sponsor confirmed

# [`Ownable` library is redundant](https://github.com/code-423n4/2022-01-trader-joe-findings/issues/241) 

# Handle

WatchPug


# Vulnerability details

https://github.com/code-423n4/2022-01-trader-joe/blob/a1579f6453bc4bf9fb0db9c627beaa41135438ed/contracts/LaunchEvent.sol#L19-L19

```solidity
contract LaunchEvent is Ownable {
```

The `LaunchEvent.sol` contract never utilized `onlyOwner` / `owner()` or any other features provided by the `Ownable` library.

Therefore, `is Ownable` can be removed.


