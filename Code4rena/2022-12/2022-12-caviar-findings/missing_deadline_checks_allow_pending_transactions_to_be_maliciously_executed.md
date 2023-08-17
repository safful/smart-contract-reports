## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- selected for report
- M-01

# [Missing deadline checks allow pending transactions to be maliciously executed](https://github.com/code-423n4/2022-12-caviar-findings/issues/28) 

# Lines of code

https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L63
https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L107
https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L147
https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L182
https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L275
https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L294
https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L310
https://github.com/code-423n4/2022-12-caviar/blob/0212f9dc3b6a418803dbfacda0e340e059b8aae2/src/Pair.sol#L323


# Vulnerability details

# Missing deadline checks allow pending transactions to be maliciously executed


### Summary
The `Pair` contract does not allow users to submit a deadline for their action. This missing feature enables pending transactions to be maliciously executed at a later point.

### Detailed description
AMMs should provide their users with an option to limit the execution of their pending actions, such as swaps or adding and removing liquidity. The most common solution is to include a deadline timestamp as a parameter (for example see [Uniswap V2](https://github.com/Uniswap/v2-periphery/blob/0335e8f7e1bd1e8d8329fd300aea2ef2f36dd19f/contracts/UniswapV2Router02.sol#L229)). If such an option is not present, users can unknowingly perform bad trades:

1. Alice wants to swap 100 fractional NFT tokens (`fTokens`) for 1 ETH and later sell the 1 ETH for 1000 DAI. She signs the transaction calling `Pair.sell` with `inputAmount = 100 fTokens` and `minOutputAmount = 0.99 ETH` to allow for some slippage.
2. The transaction is submitted to the mempool, however, Alice chose a transaction fee that is too low for miners to be interested in including her transaction in a block. The transaction stays pending in the mempool for extended periods, which could be hours, days, weeks, or even longer. 
3. When the average gas fee dropped far enough for Alice's transaction to become interesting again for miners to include it, her swap will be executed. In the meantime, the price of `ETH` could have drastically changed. She will still at least get `0.99 ETH` due to `minOutputAmount`, but the `DAI` value of that output might be significantly lower. She has unknowingly performed a bad trade due to the pending transaction she forgot about.

An even worse way this issue can be maliciously exploited is through MEV:
1. The swap transaction is still pending in the mempool. Average fees are still too high for miners to be interested in it. The price of `fToken` has gone up significantly since the transaction was signed, meaning Alice would receive a lot more `ETH` when the swap is executed. But that also means that her `minOutputAmount` value is outdated and would allow for significant slippage.
2. A MEV bot detects the pending transaction. Since the outdated `minOutputAmount` now allows for high slippage, the bot sandwiches Alice, resulting in significant profit for the bot and significant loss for Alice.

The affected functions in `Pair.sol` are:
- `add()`
- `remove()`
- `buy()`
- `sell()`
- `nftAdd()`
- `nftRemove()`
- `nftBuy()`
- `nftSell()`

### Recommended mitigation
Introduce a `deadline` parameter to the mentioned functions.


### A word on the severity
Categorizing this issue into medium versus high was not immediately obvious. I came to the conclusion that this is a high-severity issue for the following reason:

I run an arbitrage MEV bot myself, which also tracks pending transactions in the mempool, though for another reason than the one mentioned in this report. There is a *significant* amount of pending and even dropped transactions: over `200,000` transactions that are older than one month. These transactions do all kinds of things, from withdrawing from staking contracts to sending funds to CEXs and also performing swaps on DEXs like Uniswap. This goes to show that this issue will in fact be very real, there will be very old pending transactions wanting to perform trades without a doubt. And with the prevalence of advanced MEV bots, these transactions will be exploited as described in the second example above, leading to losses for Caviar's users.

### PoC
Omitted in this case, since the exploit is solely based on the fact that there is no limit on how long a transaction is allowed to be pending, which can be clearly seen when looking at the mentioned functions.