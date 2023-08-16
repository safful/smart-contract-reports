## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Repeated external calls](https://github.com/code-423n4/2022-01-insure-findings/issues/304) 

# Handle

pauliax


# Vulnerability details

## Impact
Avoid repeated external calls, e.g. here token balanceOf is queried 4 times:
```solidity
if (
    ...
    balance < IERC20(token).balanceOf(address(this))
) {
    uint256 _redundant = IERC20(token).balanceOf(address(this)) - balance;
    ...
} else if (IERC20(_token).balanceOf(address(this)) > 0) {
    IERC20(_token).safeTransfer(
        _to,
        IERC20(_token).balanceOf(address(this))
    );
}
```
You should query it only once and then use the cached value as it doesn't change between the statements.

