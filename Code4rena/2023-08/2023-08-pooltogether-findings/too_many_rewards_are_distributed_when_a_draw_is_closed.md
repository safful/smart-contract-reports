## Tags

- bug
- 3 (High Risk)
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- H-01

# [Too many rewards are distributed when a draw is closed](https://github.com/code-423n4/2023-08-pooltogether-findings/issues/139) 

# Lines of code

https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngRelayAuction.sol#L178-L184
https://github.com/GenerationSoftware/pt-v5-draw-auction/blob/f1c6d14a1772d6609de1870f8713fb79977d51c1/src/RngRelayAuction.sol#L154-L157
https://github.com/GenerationSoftware/pt-v5-prize-pool/blob/26557afa439934afc080eca6165fe3ce5d4b63cd/src/PrizePool.sol#L366
https://github.com/GenerationSoftware/pt-v5-prize-pool/blob/26557afa439934afc080eca6165fe3ce5d4b63cd/src/abstract/TieredLiquidityDistributor.sol#L374


# Vulnerability details

## Impact
A relayer completes a prize pool draw by calling `rngComplete` in `RngRelayAuction.sol`. This method closes the prize pool draw with the relayed random number and distributes the rewards to the RNG auction recipient and the RNG relay auction recipient. These rewards are calculated based on a fraction of the prize pool reserve rather than an actual value.

However, the current reward calculation mistakenly includes an extra `reserveForOpenDraw` amount just after the draw has been closed. Therefore the fraction over which the rewards are being calculated includes tokens that have not been added to the reserve and will actually only be added to the reserve when the next draw is finalised. As a result, the reward recipients are rewarded too many tokens. 

## Proof of Concept
Before deciding whether or not to relay an auction result, a bot can call `computeRewards` to calculate how many rewards they'll be getting based on the size of the reserve, the state of the auction and the reward fraction of the RNG auction recipient:

```
  function computeRewards(AuctionResult[] calldata __auctionResults) external returns (uint256[] memory) {
    uint256 totalReserve = prizePool.reserve() + prizePool.reserveForOpenDraw();
    return _computeRewards(__auctionResults, totalReserve);
  }
```

Here, the total reserve is calculated as the sum of the current reserve and and amount of new tokens that will be added to the reserve once the currently open draw is closed. This method is correct and correctly calculates how many rewards should be distributed when a draw is closed.

A bot can choose to close the draw by calling `rngComplete` (via a relayer), at which point the rewards are calculated and distributed. Below is the interesting part of this method:

```
    uint32 drawId = prizePool.closeDraw(_randomNumber);

    uint256 futureReserve = prizePool.reserve() + prizePool.reserveForOpenDraw();
    uint256[] memory _rewards = RewardLib.rewards(auctionResults, futureReserve);
```

As you can see, the draw is first closed and then the future reserve is used to calculate the rewards that should be distributed. However, when `closeDraw` is called on the pool, the `reserveForOpenDraw` for the previously open draw is added to the existing reserves. So `reserve()` is now equal to the `totalReserve` value in the earlier call to `computeRewards`. By including `reserveForOpenDraw()` when computing the actual reward to be distributed we've accidentally counted the tokens that are only going to be added in when the next draw is closed. So now the rewards distribution calculation includes the pending reserves for 2 draws rather than 1.

## Tools Used
Manual review

## Recommended Mitigation Steps
When distributing rewards in the call to `rngComplete`, the rewards should not be calculated with the new value of `reserveForOpenDraw` because the previous `reserveForOpenDraw` value has already been added to the reserves when `closeDraw` is called on the prize pool. Below is a suggested diff:

```
diff --git a/src/RngRelayAuction.sol b/src/RngRelayAuction.sol
index 8085169..cf3c210 100644
--- a/src/RngRelayAuction.sol
+++ b/src/RngRelayAuction.sol
@@ -153,8 +153,8 @@ contract RngRelayAuction is IRngAuctionRelayListener, IAuction {
 
     uint32 drawId = prizePool.closeDraw(_randomNumber);
 
-    uint256 futureReserve = prizePool.reserve() + prizePool.reserveForOpenDraw();
-    uint256[] memory _rewards = RewardLib.rewards(auctionResults, futureReserve);
+    uint256 reserve = prizePool.reserve();
+    uint256[] memory _rewards = RewardLib.rewards(auctionResults, reserve);
 
     emit RngSequenceCompleted(
       _sequenceId,

```


## Assessed type

Math