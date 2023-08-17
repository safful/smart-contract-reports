## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-19

# [`_handleDeposit` and `_handleWithdraw` do not account for tokens with decimals higher than 18](https://github.com/code-423n4/2022-12-tigris-findings/issues/533) 

# Lines of code

https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/Trading.sol#L650
https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/Trading.sol#L675


# Vulnerability details

## Impact 

In `Trading.sol` a [deposit](https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/Trading.sol#L675) or [withdrawal](https://github.com/code-423n4/2022-12-tigris/blob/main/contracts/Trading.sol#L700) of tokens with decimals higher than 18 will always revert. 

This is the case e.g. for `NEAR` which is divisible into 10e24 `yocto` 

## Proof of Concept

Change [00.Mocks.js#L33](https://github.com/code-423n4/2022-12-tigris/blob/main/deploy/test/00.Mocks.js#L33) to:

```
args: ["USDC", "USDC", 24, deployer, ethers.utils.parseUnits("1000", 24)]
```

Then in [07.Trading.js](https://github.com/code-423n4/2022-12-tigris/blob/main/test/07.Trading.js):

```
Opening and closing a position with tigUSD output
Opening and closing a position with <18 decimal token output
```

are going to fail with:
```
Error: VM Exception while processing transaction: reverted with panic code 0x11 (Arithmetic operation underflowed or overflowed outside of an unchecked block)
```

## Tools Used

Visual Studio Code

## Recommended Mitigation Steps

Update calculations in the contract to account for tokens with decimals higher than 18.
