## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-01

# [Malicious actor can win auction unfavorably to the protocol by block stuffing](https://github.com/code-423n4/2023-05-venus-findings/issues/525) 

# Lines of code

https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Shortfall/Shortfall.sol#L158-L202
https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Shortfall/Shortfall.sol#L467-L470
https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Shortfall/Shortfall.sol#L213


# Vulnerability details

## Vulnerability Details

When protocol’s bad debt is auctioned off with 10% incentive at the beginning. A user who gives the best bid, wins. The auction ends when at least one account placed a bid, and current block number is bigger than `nextBidderBlockLimit`:

```jsx
function closeAuction(address comptroller) external nonReentrant {
        Auction storage auction = auctions[comptroller];

        require(_isStarted(auction), "no on-going auction");
        require(
            block.number > auction.highestBidBlock + nextBidderBlockLimit && auction.highestBidder != address(0),
            "waiting for next bidder. cannot close auction"
        );
```

`nextBidderBlockLimit` is set to 10 in the initializer, which means that other users have only 30 seconds to place better bid. Now, this is a serious problem, because stuffing whole block with dummy transactions is very cheap on Binance Smart Chain. According to [https://www.cryptoneur.xyz/en/gas-fees-calculator](https://www.cryptoneur.xyz/en/gas-fees-calculator) 15M gas - whole block - costs 14$~15$ on BSC. This makes a malicious user occasion to cheaply prohibit other users to overbid them, winning the auction at the least favorable price for the protocol. Because BSC is centralized blockchain, there are no private mempools and bribes directly to the miners (like in FlashBots), hence other users are very limited concerning the prohibitive actions.

## Impact

The protocol overpays for bad debt, loosing value

## Proof of Concept

1. Pool gathered 100’000$ bad debt and it’s eligible for auction
2. A malicious user frontruns others and places first bid with the least possible amount (bad debt + 10% incentive).
3. The user sends dozens of dummy transactions with increased gas price, only to fill up whole block space for 11 blocks
4. At the end, the user sends a transaction to close auction, getting the bad debt + 10% incentive. 

## Tools Used

Manual analysis

## Recommended Mitigation Steps

There are at least three options to resolve this issue:

1. make he bidding window much higher at the beginning, like 1000 blocks
2. make bidding window very high at the beginning, decreasing it, the more attractive the new bid is
3. make bidding window dependent on the money at stake, to disincentivize block stuffing


## Assessed type

Other