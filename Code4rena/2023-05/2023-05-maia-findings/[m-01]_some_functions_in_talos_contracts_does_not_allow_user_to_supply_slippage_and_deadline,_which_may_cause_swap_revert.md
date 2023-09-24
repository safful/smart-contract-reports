## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-20

# [[M-01] Some functions in Talos contracts does not allow user to supply slippage and deadline, which may cause swap revert](https://github.com/code-423n4/2023-05-maia-findings/issues/504) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L269
https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/TalosStrategyVanilla.sol#L148
https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L147
https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L208
https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L359


# Vulnerability details

## Impact
In the following functions except `TalosBaseStrategy.redeem()`, the minimum slippage is still hardcoded to 0, not allowing user to specify their own slippage parameters. This can expose users to sandwich attacks due to unlimited slippage.

Additionally, it also does not allow users to supply their own deadline as the `deadline` parameter is simply passed in as current `block.timestamp` in which transaction occurs. This effectively means that transaction has no deadline, which means that swap transaction may be included anytime by validators and remain pending in mempool, potentially exposing users to sandwich attacks by attackers or MEV bots. 

- `TalosBaseStrategy.redeem()` [Link](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L269)
- `TalosStrategyVanilla._compoundFees()` [Link](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/TalosStrategyVanilla.sol#L148)
- `TalosBaseStrategy.init()` [Link](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L147)
- `TaloseBaseStrategy.deposit()` [Link](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L208)
- `TaloseBaseStrategy._withdrawAll()` [Link](https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/base/TalosBaseStrategy.sol#L359)

## Proof of Concept

Consider the following scenario:
1. Alice wants to swap 30 BNB token for 1 BNB and later sell the 1 BNB for 300 DAI. She signs the transaction calling `TalosBaseStrategy.redeem()` with inputAmount = 30 vBNB and `amountOutmin` = 0.99 BNB (1$ slippage).

<br/>

2. The transaction is submitted to the mempool, however, Alice chose a transaction fee that is too low for validators to be interested in including her transaction in a block. The transaction stays pending in the mempool for extended periods, which could be hours, days, weeks, or even longer.

<br/>

3. When the average gas fee dropped far enough for Alice's transaction to become interesting again for miners to include it, her swap will be executed. In the meantime, the price of BNB could have drastically decreased. She will still at least get 0.99 BNB due to `amountOutmin`, but the DAI value of that output might be significantly lower. She has unknowingly performed a bad trade due to the pending transaction she forgot about.

An even worse way this issue can be maliciously exploited is through MEV:

1. The swap transaction is still pending in the mempool. Average fees are still too high for validators to be interested in it. The price of BNB has gone up significantly since the transaction was signed, meaning Alice would receive a lot more ETH when the swap is executed. But that also means that her `minOutput` value is outdated and would allow for significant slippage.

<br/>

2. A MEV bot detects the pending transaction. Since the outdated `minOut` now allows for high slippage, the bot sandwiches Alice, resulting in significant profit for the bot and significant loss for Alice.

The above scenario could be made worst for other functions where slippage is not allowed to be user-specified, where combined with a lack of deadline check, MEV bots can simply immediately sandwich users.

## Tools Used
Manual Analysis

## Recommendation
Allow users to supply their own slippage and deadline parameter within the stated functions. The deadline modifier can than can be checked via a modifier or check, which has already been implemented via the `checkDeadline()` modifier.








## Assessed type

Timing