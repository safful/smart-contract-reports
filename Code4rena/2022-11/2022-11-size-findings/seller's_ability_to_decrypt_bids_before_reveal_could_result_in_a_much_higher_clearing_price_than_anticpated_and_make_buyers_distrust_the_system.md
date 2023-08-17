## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- selected for report
- M-05

# [Seller's ability to decrypt bids before reveal could result in a much higher clearing price than anticpated and make buyers distrust the system](https://github.com/code-423n4/2022-11-size-findings/issues/194) 

# Lines of code

https://github.com/code-423n4/2022-11-size/blob/706a77e585d0852eae6ba0dca73dc73eb37f8fb6/src/SizeSealed.sol#L140-L142


# Vulnerability details

## Impact
Bids are encrypted with the seller's public key. This makes that the seller can see all the bids before they reveal and finalize the auction. 
If the bids are unsatisfactory the seller can just decide not to honor the auction and not reveal their private key. Even worse, the seller, knowing the existing bids, can choose to fill the auction just beneath the optimal price increasing the clearing price. 

Although there is a [check](https://github.com/code-423n4/2022-11-size/blob/706a77e585d0852eae6ba0dca73dc73eb37f8fb6/src/SizeSealed.sol#L140-L142) in the `bid()` function where the bidder cannot submit a bid with the same address as the one used while creating the auction it is trivial to use another address to submit bids.

Whether this results in a net increase of the total amount received is not a limiting factor as they might very well be happy with a lower total amount of quote tokes if it means selling less of the base tokens. 

Ultimately this means the reserve price for the auction is not respected which would make bidders distrust the system. 
Bidders that don't realize this impact stand to pay a much higher price than anticipated, hence the HIGH risk rating.

## Proof of Concept
Following test shows the seller knowing the highest price can force the clearing price (`lowestQuote`) to be 2e6 in stead of what would be 1e6. 
`sellerIncreases` can be set to true or false to simulate both cases.
```solidity
    function testAuctionSellerIncreasesClearing() public {
        bool sellerIncreases = true;
        (uint256 sellerBeforeQuote, uint256 sellerBeforeBase) = seller.balances();
        uint256 aid = seller.createAuction(
            baseToSell, reserveQuotePerBase, minimumBidQuote, startTime, endTime, unlockTime, unlockEnd, cliffPercent
        );
        bidder1.setAuctionId(aid);
        bidder2.setAuctionId(aid);
        bidder1.bidOnAuctionWithSalt(9 ether, 18e6, "bidder1");
        bidder2.bidOnAuctionWithSalt(1 ether, 1e6, "bidder2");
        uint256[] memory bidIndices = new uint[](2);
        bidIndices[0] = 0;
        bidIndices[1] = 1;
        uint128 expectedClearingPrice = 1e6;
        if (sellerIncreases){
            //seller's altnerate wallet
            bidder3.setAuctionId(aid);
            bidder3.bidOnAuctionWithSalt(1 ether, 2e6, "seller");
            bidIndices = new uint[](3);
            bidIndices[0] = 0;
            bidIndices[1] = 2;
            bidIndices[2] = 1;
            expectedClearingPrice = 2e6;
        }          
        vm.warp(endTime + 1);
        seller.finalize(bidIndices, 1 ether, expectedClearingPrice);
        AuctionData memory data = auction.getAuctionData(aid);
        emit log_named_uint("lowestBase", data.lowestBase);
        emit log_named_uint("lowestQuote", data.lowestQuote);
    }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
A possible mitigation would be to introduce a 2 step reveal where the bidders also encrypt their bid with their own private key and only reveal their key after the seller has revealed theirs. 
This would however create another attack vector where bidders try and game the auction and only reveal their bid(s) when and if the result would be in their best interest. 
This in turn could be mitigated by bidders losing (part of) their quote tokens when not revealing their bids.