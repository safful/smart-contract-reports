## Tags

- bug
- 2 (Med Risk)
- grade-b
- satisfactory
- selected for report
- sponsor acknowledged
- M-01

# [`QuestFactory` is suspicious of the reorg attack](https://github.com/code-423n4/2023-01-rabbithole-findings/issues/661) 

# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L75
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/QuestFactory.sol#L108


# Vulnerability details

## Description

The `createQuest` function deploys a quest contract using the `create`, where the address derivation depends only on the `QuestFactory` nonce. 

At the same time, some of the chains (Polygon, Optimism, Arbitrum) to which the `QuestFactory` will be deployed are suspicious of the reorg attack.

- https://polygonscan.com/blocks_forked

![](https://i.imgur.com/N8tDUVX.png)

Here you may be convinced that the Polygon has in practice subject to reorgs. Even more, the reorg on the picture is 1.5 minutes long. So, it is quite enough to create the quest and transfer funds to that address, especially when someone uses a script, and not doing it by hand.

Optimistic rollups (Optimism/Arbitrum) are also suspect to reorgs since if someone finds a fraud the blocks will be reverted, even though the user receives a confirmation and already created a quest.

## Attack scenario

Imagine that Alice deploys a quest, and then sends funds to it. Bob sees that the network block reorg happens and calls `createQuest`. Thus, it creates `quest` with an address to which Alice sends funds. Then Alices' transactions are executed and Alice transfers funds to Bob's controlled quest. 

## Impact

If users rely on the address derivation in advance or try to deploy the wallet with the same address on different EVM chains, any funds sent to the wallet could potentially be withdrawn by anyone else. All in all, it could lead to the theft of user funds.

## Recommended Mitigation Steps

Deploy the quest contract via `create2` with `salt` that includes `msg.sender` and `rewardTokenAddress_`.


