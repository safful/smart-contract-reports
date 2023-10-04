## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report

# [Attacker can extract value from pool by sandwiching herself at `swapAll` during `close`](https://github.com/code-423n4/2023-09-goodentry-mitigation-findings/issues/60) 

# Lines of code

https://github.com/GoodEntry-io/ge/blob/c7c7de57902e11e66c8186d93c5bb511b53a45b8/contracts/PositionManager/OptionsPositionManager.sol#L454


# Vulnerability details

Attacker can drain the lending pool by leveraging two facts:

1. `swapAll` allows 1% slippage
2. There is no Health Factor check after `close`.

Alice and Bob are good friends, the steps are (in one single tx):

1. Alice deposits 10000 USDT and borrows 7000$ worth of TR.
2. Bob buys ETH at AMM to push up the price to oracle + 1%.
3. Alice `close` but only repays 1 wei debt. The real intention is to swap from USDT collateral to ETH collateral.
4. Bob sells ETH at AMM to pull down the price to oracle - 1%.
5. Alice `close` but only repays 1 wei debt to swap to USDT collateral.
6. Repeat
7. Alice has 0 collateral and Bob gains 10000 USDT by sandwiching.

By continues sandwiching Alice, Bob can extract value from the pool. A simple mitigation is to add a HF check after each swap.



## Assessed type

Context