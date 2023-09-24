## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-33

# [`unstakeAndWithdraw` inside `BoostAggregator` could lose pendingRewards in certain case](https://github.com/code-423n4/2023-05-maia-findings/issues/287) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/boost-aggregator/BoostAggregator.sol#L109-L136
https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/boost-aggregator/BoostAggregator.sol#L86


# Vulnerability details

## Impact
When `BoosAggregator`'s `unstakeAndWithdraw` triggered, it will try to unstake uniswap NFT position token from staker, get the pending rewards and if condition met, update strategy and protocol rewards accounting and claim the rewards for strategy and finally withdraw NFT position token from staker. However, if `pendingRewards` lower than `DIVISIONER` the accounting will not happened and can cause reward loss.

## Proof of Concept

Inside `unstakeAndWithdraw`, if `pendingRewards` lower than `DIVISIONER`, the accounting update for `protocolRewards` and claim rewards for strategy not happened :

https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/boost-aggregator/BoostAggregator.sol#L109-L136

```solidity
    function unstakeAndWithdraw(uint256 tokenId) external {
        address user = tokenIdToUser[tokenId];
        if (user != msg.sender) revert NotTokenIdOwner();

        // unstake NFT from Uniswap V3 Staker
        uniswapV3Staker.unstakeToken(tokenId);

        uint256 pendingRewards = uniswapV3Staker.tokenIdRewards(tokenId) - tokenIdRewards[tokenId];

        if (pendingRewards > DIVISIONER) {
            uint256 newProtocolRewards = (pendingRewards * protocolFee) / DIVISIONER;
            /// @dev protocol rewards stay in stake contract
            protocolRewards += newProtocolRewards;
            pendingRewards -= newProtocolRewards;

            address rewardsDepot = userToRewardsDepot[user];
            if (rewardsDepot != address(0)) {
                // claim rewards to user's rewardsDepot
                uniswapV3Staker.claimReward(rewardsDepot, pendingRewards);
            } else {
                // claim rewards to user
                uniswapV3Staker.claimReward(user, pendingRewards);
            }
        }

        // withdraw rewards from Uniswap V3 Staker
        uniswapV3Staker.withdrawToken(tokenId, user, "");
    }
```

However, when the token staked again via `BoosAggregator` by sending the NFT position back, the `tokenIdRewards` rewards updated, so the previous unaccounted rewards will be loss :

https://github.com/code-423n4/2023-05-maia/blob/main/src/talos/boost-aggregator/BoostAggregator.sol#L79-L93

```solidity
    function onERC721Received(address, address from, uint256 tokenId, bytes calldata)
        external
        override
        onlyWhitelisted(from)
        returns (bytes4)
    {
        // update tokenIdRewards prior to staking
        tokenIdRewards[tokenId] = uniswapV3Staker.tokenIdRewards(tokenId);
        // map tokenId to user
        tokenIdToUser[tokenId] = from;
        // stake NFT to Uniswap V3 Staker
        nonfungiblePositionManager.safeTransferFrom(address(this), address(uniswapV3Staker), tokenId);

        return this.onERC721Received.selector;
    }
```

## Tools Used

Manual review

## Recommended Mitigation Steps

Two things can be done here, either just claim reward to strategy without taking the protocol fee, or take the amount fully for protocol.


## Assessed type

Error