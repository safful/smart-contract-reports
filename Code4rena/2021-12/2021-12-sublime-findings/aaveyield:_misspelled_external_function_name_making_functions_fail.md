## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed

# [AaveYield: Misspelled external function name making functions fail](https://github.com/code-423n4/2021-12-sublime-findings/issues/42) 

# Handle

0xngndev


# Vulnerability details

## Impact

In `AaveYield.sol` the functions:

- `liquidityToken`
- `_withdrawETH`
- `_depositETH`

Make a conditional call to `IWETHGateway(wethGateway).getAWETHAddress()`

This function does not exist in the `wethGateway` contract, causing these function to fail with the error `"Fallback not allowed"`.

The function they should be calling is `getWethAddress()` without the "A".

Small yet dangerous typo.

### Mitigation Steps

Simply modify:

 `IWETHGateway(wethGateway).getAWETHAddress()`

to:

`IWETHGateway(wethGateway).getWETHAddress()`

In the functions mentioned above.

