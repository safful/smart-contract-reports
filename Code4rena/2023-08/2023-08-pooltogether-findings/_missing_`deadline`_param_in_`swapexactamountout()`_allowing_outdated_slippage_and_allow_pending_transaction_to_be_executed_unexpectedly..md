## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- M-03

# [ Missing `deadline` param in `swapExactAmountOut()` allowing outdated slippage and allow pending transaction to be executed unexpectedly.](https://github.com/code-423n4/2023-08-pooltogether-findings/issues/126) 

# Lines of code

https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/LiquidationRouter.sol#L63-L80
https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/LiquidationPair.sol#L211-L226


# Vulnerability details


## Impact

Loss of funds/tokens for the protocol, since block execution is delegated to the block validator without a hard deadline.

## Proof of Concept

The function `swapExactAmountOut()` from `LiquidationRouter.sol` and `LiquidationPair.sol` use these methods to swap tokens:

`source.liquidate(_account, tokenIn, swapAmountIn, tokenOut, _amountOut);`

and

`_liquidationPair.swapExactAmountOut(_receiver, _amountOut, _amountInMax);`

https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/LiquidationPair.sol#L211-L226

https://github.com/GenerationSoftware/pt-v5-cgda-liquidator/blob/7f95bcacd4a566c2becb98d55c1886cadbaa8897/src/LiquidationRouter.sol#L63-L80

Both methods make sure to pass slippage (minimum amount out), but miss to provide the deadline which is crucial to avoid unexpected trades/losses for users and protocol.

Without a deadline, the transaction might be left hanging in the mempool and be executed way later than the user wanted.

That could lead to users/protocol getting a worse price, because a validator can just hold onto the transaction. And when it does get around to putting the transaction in a block

One part of this change is that PoS block proposers know ahead of time if they're going to propose the next block. The validators and the entire network know who's up to bat for the current block and the next one.

This means the block proposers are known for at least 6 minutes and 24 seconds and at most 12 minutes and 48 seconds.

Further reading:
https://blog.bytes032.xyz/p/why-you-should-stop-using-block-timestamp-as-deadline-in-swaps

## Tools Used
Manual

## Recommended Mitigation Steps

Let users provide a fixed deadline as param, and also never set deadline to `block.timestamp`.



## Assessed type

Other