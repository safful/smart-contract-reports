## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-03

# [Fee on transfer tokens will not behave as expected](https://github.com/code-423n4/2023-01-timeswap-findings/issues/247) 

# Lines of code

https://github.com/code-423n4/2023-01-timeswap/blob/main/packages/v2-option/src/TimeswapV2Option.sol#L145-L148
https://github.com/code-423n4/2023-01-timeswap/blob/main/packages/v2-option/src/TimeswapV2Option.sol#L235


# Vulnerability details

## Impact
According to [Whitepaper 1.1 Permissionless](https://github.com/code-423n4/2023-01-timeswap/blob/main/whitepaper.pdf):

"In Timeswap, liquidity providers can create pools for any ERC20 pair, without permission. It is designed to be generalized and works
for any pair of tokens, at any time frame, and at any market state ...

If fee on transfer token(s) is/are entailed, it will specifically make `mint()` and `swap()` revert in TimeswapV2Option.sol when checking if the token0 or token1 balance target is achieved.

## Proof of Concept
[File: TimeswapV2Option.sol#L144-L148](https://github.com/code-423n4/2023-01-timeswap/blob/main/packages/v2-option/src/TimeswapV2Option.sol#L144-L148)

```solidity
        // check if the token0 balance target is achieved.
        if (token0AndLong0Amount != 0) Error.checkEnough(IERC20(token0).balanceOf(address(this)), currentProcess.balance0Target);

        // check if the token1 balance target is achieved.
        if (token1AndLong1Amount != 0) Error.checkEnough(IERC20(token1).balanceOf(address(this)), currentProcess.balance1Target);
```
[File: TimeswapV2Option.sol#L234-L235](https://github.com/code-423n4/2023-01-timeswap/blob/main/packages/v2-option/src/TimeswapV2Option.sol#L234-L235)

```solidity
        // check if the token0 or token1 balance target is achieved.
        Error.checkEnough(IERC20(param.isLong0ToLong1 ? token1 : token0).balanceOf(address(this)), param.isLong0ToLong1 ? currentProcess.balance1Target : currentProcess.balance0Target);
```
[File: Error.sol#L148-L153](https://github.com/code-423n4/2023-01-timeswap/blob/main/packages/v2-library/src/Error.sol#L148-L153)

```solidity
    /// @dev Reverts when token amount not received.
    /// @param balance The balance amount being subtracted.
    /// @param balanceTarget The amount target.
    function checkEnough(uint256 balance, uint256 balanceTarget) internal pure {
        if (balance < balanceTarget) revert NotEnoughReceived(balance, balanceTarget);
    }
```
As can be seen from the code blocks above, `checkEnough()` is meant to be reverting when token amount has not been received. But in the case of deflationary tokens, the error is going to be thrown even though the token amount has been received due to the fee factor making `balance < balanceTarget`, i.e the contract balance of token0 and/or token1 always less than `currentProcess.balance0Target` or `currentProcess.balance1Target`.

## Tools Used
Manual inspection

## Recommended Mitigation Steps
Consider:

1. whitelisting token0 and token1 ensuring no fee-on-transfer token is allowed when deploying a new Timeswap V2 Option pair contract, or
2. calculating the balance before and after the [transfer to the recipient](https://github.com/code-423n4/2023-01-timeswap/blob/main/packages/v2-option/src/TimeswapV2Option.sol#L220) during the process, and use the difference between those two balances as the amount received rather than using the input amount (`token0AndLong0Amount` or `token1AndLong1Amount`) if deflationary token is going to be allowed in the protocol.
