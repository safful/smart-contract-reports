## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- M-04

# [Users may not claim Erc1155 rewards when the Quest has ended](https://github.com/code-423n4/2023-01-rabbithole-findings/issues/528) 

# Lines of code

https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc1155Quest.sol#L60
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L114
https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc1155Quest.sol#L41-L43


# Vulnerability details

## Impact
Unlike Erc20Quest.sol, owner of Erc1155Quest.sol is going to withdraw the remaining tokens from the contract when `block.timestamp == endTime` without deducting the `unclaimedTokens`. As a result, users will be denied of service when attempting to call the inherited `claim()` from Quest.sol. 

## Proof of Concept
As can be seen from the code block below, when the Quest time has ended, `withdrawRemainingTokens()` is going to withdraw the remaining tokens from the contract on line 60:

[File: Erc1155Quest.sol#L52-L63](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc1155Quest.sol#L52-L63)

```solidity
    /// @dev Withdraws the remaining tokens from the contract. Only able to be called by owner
    /// @param to_ The address to send the remaining tokens to
    function withdrawRemainingTokens(address to_) public override onlyOwner {
        super.withdrawRemainingTokens(to_);
        IERC1155(rewardToken).safeTransferFrom(
            address(this),
            to_,
            rewardAmountInWeiOrTokenId,
60:            IERC1155(rewardToken).balanceOf(address(this), rewardAmountInWeiOrTokenId),
            '0x00'
        );
    }
```
When a user tries to call `claim()` below, line 114 is going to internally invoke `_transferRewards()`:

[File: Quest.sol#L94-L118](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Quest.sol#L94-L118)

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
114:        _transferRewards(totalRedeemableRewards);
        redeemedTokens += redeemableTokenCount;

        emit Claimed(msg.sender, totalRedeemableRewards);
    }
```
`safeTransferFrom()` is going to revert on line 42 because the token balance of the contract is now zero. i.e. less than `amount_`:

[File: Erc1155Quest.sol#L39-L43](https://github.com/rabbitholegg/quest-protocol/blob/8c4c1f71221570b14a0479c216583342bd652d8d/contracts/Erc1155Quest.sol#L39-L43)

```solidity
    /// @dev Transfers the reward token `rewardAmountInWeiOrTokenId` to the msg.sender
    /// @param amount_ The amount of reward tokens to transfer
    function _transferRewards(uint256 amount_) internal override {
42:        IERC1155(rewardToken).safeTransferFrom(address(this), msg.sender, rewardAmountInWeiOrTokenId, amount_, '0x00');
    }
```
## Tools Used
Manual inspection

## Recommended Mitigation Steps
Consider refactoring `withdrawRemainingTokens()` as follows:

(Note: The contract will have to separately import {QuestFactory} from './QuestFactory.sol' and initialize `questFactoryContract`.

```diff
+    function receiptRedeemers() public view returns (uint256) {
+        return questFactoryContract.getNumberMinted(questId);
+    }

    function withdrawRemainingTokens(address to_) public override onlyOwner {
        super.withdrawRemainingTokens(to_);

+        uint unclaimedTokens = (receiptRedeemers() - redeemedTokens)
+        uint256 nonClaimableTokens = IERC1155(rewardToken).balanceOf(address(this), rewardAmountInWeiOrTokenId) - unclaimedTokens;
        IERC1155(rewardToken).safeTransferFrom(
            address(this),
            to_,
            rewardAmountInWeiOrTokenId,
-            IERC1155(rewardToken).balanceOf(address(this), rewardAmountInWeiOrTokenId),
+            nonClaimableTokens,
            '0x00'
        );
    }
``` 