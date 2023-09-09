## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor confirmed
- H-11

# [Protocol loses fees because highWaterMark is updated every time someone deposit, withdraw, mint](https://github.com/code-423n4/2023-01-popcorn-findings/issues/70) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L138
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L215
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L480-L499
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L447-L460


# Vulnerability details

## Impact
Protocol loses fees because highWaterMark is updated every time someone deposit, withdraw, mint.

## Proof of Concept
This bug is related to the fees accruing design. It was disscussed with the sponsor in order to understand how it should work.

Protocol has such thing as performance fee. Actually this is fee from accrued yields. If user deposited X assets and after some time he can withdraw X+Y assets for that minted amount of shares, that means that startegy has earned some Y amount of yields. Then protocol is able to get some part of that Y amount as a performance fee.

`takeFees` [modifier](https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L480-L494) is responsible for taking fees.
It calls `accruedPerformanceFee` function to calculate fees amount.
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L447-L460
```solidity
    function accruedPerformanceFee() public view returns (uint256) {
        uint256 highWaterMark_ = highWaterMark;
        uint256 shareValue = convertToAssets(1e18);
        uint256 performanceFee = fees.performance;


        return
            performanceFee > 0 && shareValue > highWaterMark
                ? performanceFee.mulDiv(
                    (shareValue - highWaterMark) * totalSupply(),
                    1e36,
                    Math.Rounding.Down
                )
                : 0;
    }
``` 
As you can see, protocol has such variable as `highWaterMark`. This variable actually should store `convertToAssets(1e18)` amount at the time when last fee were accrued or after first deposit.
Then after some time when strategy earned some yields, `convertToAssets(1e18)` will return more assets than `highWaterMark`, so protocol will take fees.

But currently updating of `highWaterMark` is done incorrectly.
Deposit, mint, withdraw function [use `syncFeeCheckpoint` modifier](https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L138).
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L496-L499
```solidity
    modifier syncFeeCheckpoint() {
        _;
        highWaterMark = convertToAssets(1e18);
    }
```
This modifier will update `highWaterMark` to current assets amount that you can receive for 1e18 of shares.
That means that every time when deposit, mint, withdraw is called, `highWaterMark` is updated to the new state, so protocol doesn't track yield progress anymore.

In case if protocol accrued some performance fees, which can be possible if noone called deposit, withdraw, mint for a long time, then anyone can frontrun `takeFees` and deposit small amount of assets in order to update `highWaterMark`, so protocol will not get any fees.
## Tools Used
VsCode
## Recommended Mitigation Steps
I believe that you need to store `highWaterMark = convertToAssets(1e18)` at the time of first deposit, or when totalShares is 0,
this will be the value that protocol started with
and then at time, when takefee was called you can calculate current convertToAssets(1e18)
in case if it's bigger, than previous stored, then you can mint fees for protocol and update highWaterMark to current value.