## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- satisfactory
- selected for report
- sponsor confirmed
- M-14

# [placeBid() Possible participation in auctions that have been modified](https://github.com/code-423n4/2023-05-venus-findings/issues/62) 

# Lines of code

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Shortfall/Shortfall.sol#L158


# Vulnerability details

## Impact
`placeBid()` Lack of checking if auction are restarted and participating in auction that are not expected by the user may result in the user losing funds

## Proof of Concept
When the user makes a bid, simply pass in `comptroller` and `bidBps` with the following code:

```solidity
    function placeBid(address comptroller, uint256 bidBps) external nonReentrant {
        Auction storage auction = auctions[comptroller];

        require(_isStarted(auction), "no on-going auction");
        require(!_isStale(auction), "auction is stale, restart it");
        require(bidBps <= MAX_BPS, "basis points cannot be more than 10000");
```

Because `comptroller` corresponds to the `Auction` there are two cases that will generate new `Auction` (the new one and the old one may have completely different types and amounts)

1. If the first auction takes too long and no one bids `_isStale()` you can restart the auction
2. If the auction ends and the last bid time `> nextBidderBlockLimit`, then you can restart the auction after it ends

Since bidding can be restarted, so `placeBid()` is only based on `comptroller`, it may be possible to participate in the old `Auction` but end up participating in the new `Auction`, as the old and new `Auction` may be very different, resulting in the user losing money.

For example, the following scenario

1. Alice learns about the auction in the UI, auctions = {type=LARGE_RISK_FUND,debt=100,seizedRiskFund=100} and submits it to participate in the auction
2. When alice commits, her transaction will exist in memorypool

Note: Because alice needs to stay in the UI interface for some time, or because of the GAS price and block size, alice transaction is delayed, resulting in a much higher possibility of preemptive execution in step 3

3. The auction is restarted due to any of the following situations (before alice's task is executed)

a) auction `>waitForFirstBidder` leads to `_isStale()`, bob executes restarted auction, restarted auction debt increases e.g.: auction = {type=LARGE_POOL_DEBT,debt=100000,seizedRiskFund=100}

b) The auction ends `> nextBidderBlockLimit`, bob restarts the auction, the restarted auction `seizedRiskFund` becomes small, like 0, the debt is already very high as
auction = {type=LARGE_POOL_DEBT,debt=100,seizedRiskFund=0}, there may also be still `LARGE_RISK_FUND`

4. alice's turn to execute the transaction, this time the new auction and alice expected has been much worse, but the transaction will still be executed. The result is that alice may pay a lot of debt, but get very little `seizedRiskFund`.

So, we need to add a restriction to `placeBid()` to ensure that the auction has not been restarted when the transaction is executed.

A simple way to do this is to add the parameter : `auctionStartBlock`, and compare it to the `auction.startBlock` of the current transaction, if the two are different, then the auction has been restarted and the transaction revert

## Tools Used

## Recommended Mitigation Steps

```solidity
-   function placeBid(address comptroller, uint256 bidBps) external nonReentrant {
+   function placeBid(address comptroller, uint256 bidBps , uint256 auctionStartBlock) external nonReentrant {    
        Auction storage auction = auctions[comptroller];

+       require(auction.startBlock == auctionStartBlock,"auction has been restarted");
        require(_isStarted(auction), "no on-going auction");
        require(!_isStale(auction), "auction is stale, restart it");
        require(bidBps <= MAX_BPS, "basis points cannot be more than 10000");
```


## Assessed type

Context