## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-08

# [Buyer on secondary NFT market can lose fund if they buy a NFT that is already used to claim the reward](https://github.com/code-423n4/2023-01-rabbithole-findings/issues/119) 

# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L113


# Vulnerability details

## Impact

Buyer on secondary NFT market can lose fund if they buy a NFT that is already used to claim the reward

## Proof of Concept

Let us look closely into the Quest.sol#claim function

```solidity
/// @notice Allows user to claim the rewards entitled to them
/// @dev User can claim based on the (unclaimed) number of tokens they own of the Quest
function claim() public virtual onlyQuestActive {
	if (isPaused) revert QuestPaused();

	uint[] memory tokens = rabbitHoleReceiptContract.getOwnedTokenIdsOfQuest(questId, msg.sender);

	if (tokens.length == 0) revert NoTokensToClaim();

	uint256 redeemableTokenCount = 0;
	for (uint i = 0; i < tokens.length; i++) {
		if (!isClaimed(tokens[i])) {
			redeemableTokenCount++;
		}
	}

	if (redeemableTokenCount == 0) revert AlreadyClaimed();

	uint256 totalRedeemableRewards = _calculateRewards(redeemableTokenCount);
	_setClaimed(tokens);
	_transferRewards(totalRedeemableRewards);
	redeemedTokens += redeemableTokenCount;

	emit Claimed(msg.sender, totalRedeemableRewards);
}
```

After the NFT is used to claim, the _setClaimed(token) is called to mark the NFT as used to prevent double claiming.

```solidity
/// @notice Marks token ids as claimed
/// @param tokenIds_ The token ids to mark as claimed
function _setClaimed(uint256[] memory tokenIds_) private {
	for (uint i = 0; i < tokenIds_.length; i++) {
		claimedList[tokenIds_[i]] = true;
	}
}
```

The NFT is also tradeable in the secondary marketplace, I woud like to make a reasonable assumption that user wants to buy the NFT because they can use the NFT to claim the reward, which means after the reward is claimed, the NFT lose value.

Consider the case below:

1. User A has 1 NFT, has he can use the NFT to claim 1 ETH reward.
2. User A place a sell order in opensea and sell the NFT for 0.9 ETH.
3. User B see the sell order and find it a good trae, he wants to buy the NFT.
4. User B submit a buy order, User A at the same time submit the claimReward transaction.
5. User A's transaction executed first, reward goes to User A, then User B transaction executed, NFT ownership goes to User B, but user B find out that the he cannot claim the reward becasue the reward is already claimed by User A.

User A can intentionally front-run User B's buy transaction by monitoring the mempool
in polygon using the service

https://www.blocknative.com/blog/polygon-mempool

Or it could be just two user submit transaction at the same and User A's claim transaction happens to execute first.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Disable NFT transfer and trade once the NFT is used to claim the reward.