## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Inconsistencies when checking if the auction is active](https://github.com/code-423n4/2021-11-malt-findings/issues/352) 

# Handle

pauliax


# Vulnerability details

## Impact
When auction.endingTime == now, function purchaseArbitrageTokens thinks that auction is still active, while isAuctionFinished and earlyExitReturn think that it has ended:
purchaseArbitrageTokens:
```solidity
  require(auction.endingTime >= now, "Auction is already over");
```
isAuctionFinished:
```solidity
  return auction.endingTime > 0 && (now >= auction.endingTime || ...);
```
earlyExitReturn:
```solidity
  if(active || block.timestamp < auctionEndTime) {
    return 0;
  }
```

## Recommended Mitigation Steps
Consider unifying it across the functions.

