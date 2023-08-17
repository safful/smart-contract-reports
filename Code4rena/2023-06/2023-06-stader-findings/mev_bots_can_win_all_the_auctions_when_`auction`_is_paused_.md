## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor disputed
- M-07

# [MEV bots can win all the auctions when `Auction` is paused ](https://github.com/code-423n4/2023-06-stader-findings/issues/226) 

# Lines of code

https://github.com/code-423n4/2023-06-stader/blob/main/contracts/Auction.sol#L62-L78


# Vulnerability details

## Impact
MEV bots may bid all the ongoing auctions before `Auction` is paused by frontrunning it with the `addBid` function. 

This can lead to bots winning all the auctions for a fraction of their price, especially auctions that are almost ended, as no one is able to bid until the contract is unpaused.

## Proof of Concept
No one will be able to bid when the contract is paused, but bots can simply front-run the `pause` by looking at the mempool:

```solidity
	function addBid(uint256 lotId) external payable override whenNotPaused {
	    // reject payments of 0 ETH
	    if (msg.value == 0) revert InSufficientETH();

	    LotItem storage lotItem = lots[lotId];
	    if (block.number > lotItem.endBlock) revert AuctionEnded();

	    uint256 totalUserBid = lotItem.bids[msg.sender] + msg.value;

	    if (totalUserBid < lotItem.highestBidAmount + bidIncrement) revert InSufficientBid();

	    lotItem.highestBidder = msg.sender;
	    lotItem.highestBidAmount = totalUserBid;
	    lotItem.bids[msg.sender] = totalUserBid;

	    emit BidPlaced(lotId, msg.sender, totalUserBid);
	}
```

**A word about severity**; the contract extends `PausableUpgradeable`:

```solidity
	contract Auction is IAuction, Initializable, AccessControlUpgradeable, PausableUpgradeable, ReentrancyGuardUpgradeable
```

Also, it has several functions that have the `whenNotPaused` modifier: 

```solidity
	function createLot(uint256 _sdAmount) external override whenNotPaused

	function addBid(uint256 lotId) external payable override whenNotPaused
```

However, it doesn't have any `pause`/`unpause` functions that are `external`/`public`, as they have `internal` visibility by default in `PausableUpgradeable`; as such, the contract can't be paused with the current implementation.

However, this may change in the future as the contract is upgradeable, so I still consider this issue valid.

## Tools Used
Manual review

## Recommended Mitigation Steps
Consider removing the `whenNotPaused` modifier from `addBid`, so ongoing auctions may be ended even when the contract is paused.


## Assessed type

MEV