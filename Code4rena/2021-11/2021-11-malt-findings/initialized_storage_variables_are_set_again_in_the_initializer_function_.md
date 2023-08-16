## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [initialized storage variables are set again in the initializer function ](https://github.com/code-423n4/2021-11-malt-findings/issues/45) 

# Handle

sabtikw


# Vulnerability details

## Impact

storage variables are initialized in the contract and overwritten in the initializer function. 

## Proof of Concept
Auction.sol L#89 L#164 auctionLength
AuctionBurnReserveSkew.sol L#25 auctionAverageLookback
MaltDataLab.sol L#69 priceTarget

## Tools Used

manual review 

## Recommended Mitigation Steps

remove initialization outside of initializer function

