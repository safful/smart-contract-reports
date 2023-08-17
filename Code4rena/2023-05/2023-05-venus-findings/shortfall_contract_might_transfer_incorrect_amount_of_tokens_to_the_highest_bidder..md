## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-09

# [ShortFall contract might transfer incorrect amount of tokens to the highest bidder.](https://github.com/code-423n4/2023-05-venus-findings/issues/222) 

# Lines of code

https://github.com/code-423n4/2023-05-venus/blob/main/contracts/Shortfall/Shortfall.sol#L248


# Vulnerability details

### Impact

There might be an incorrect amount of transfer possible if `convertibleBaseAsset` is not a token which is pegged to USD. 

### Details

There is not much information on what `convertibleBaseAsset` is supposed to be, if it is a token which is not pegged to USD then the auction process might transfer wrong amount of tokens or entirely wrong tokens.

Let’s take an example of `LARGE_RISK_FUND` type of auction for simplicity. Assuming the `convertibleBaseAsset` is not a token pegged to USD (let’s take it as BNB for this case)

Now the calculation of poolBadDebt is calculated by converting the badDebt to usd terms in the code below:

```solidity
for (uint256 i; i < marketsCount; ++i) {
            uint256 marketBadDebt = vTokens[i].badDebt();

            priceOracle.updatePrice(address(vTokens[i]));
            uint256 usdValue = (priceOracle.getUnderlyingPrice(address(vTokens[i])) * marketBadDebt) / 1e18;

            poolBadDebt = poolBadDebt + usdValue;
            auction.markets[i] = vTokens[i];
            auction.marketDebt[vTokens[i]] = marketBadDebt;
            marketsDebt[i] = marketBadDebt;
        }
```

In this case the auction properties would be as follows:

```solidity
auction.seizedRiskFund = incentivizedRiskFundBalance;
auction.startBlock = block.number;
auction.status = AuctionStatus.STARTED;
auction.highestBidder = address(0);
```

Where `incentivizedRiskFundBalance` = `poolBadDebt + ((poolBadDebt * incentiveBps) / MAX_BPS);` 

```solidity
function closeAuction(address comptroller) external nonReentrant {
        Auction storage auction = auctions[comptroller];
				// ...
        if (auction.auctionType == AuctionType.LARGE_POOL_DEBT) {
            riskFundBidAmount = auction.seizedRiskFund;
        } else {
            riskFundBidAmount = (auction.seizedRiskFund * auction.highestBidBps) / MAX_BPS;
        }

        uint256 transferredAmount = riskFund.transferReserveForAuction(comptroller, riskFundBidAmount);
        IERC20Upgradeable(convertibleBaseAsset).safeTransfer(auction.highestBidder, riskFundBidAmount); 
    }
```

When the `closeAuction`  is called the contract will transfer the `riskFundBidAmount` of `convertibleBaseAsset` to the highest bidder. Here if the `convertibleBaseAsset` token is not a token pegged to USD, it will transfer those tokens to the highest bidder, where it should have transferred the tokens that amount to that value. 

Eg:  `convertibleBaseAsset` = TokenA (price of this token is $100 per 1e18 tokens)

PoolBadDebt = 200*ie18 which should be equal to 200$ as poolbaddebt is calculated in usd

seizedRiskFund = 220*1e18

Assume the auction type is `LARGE_RISK_FUND` and highestBidBps = 10000. 

At the auction complete the tokens transferred to the highestbidder would be :

riskFundBidAmount = 220*1e18  * 10000/10000 = 220*1e18 

Actual price of tokens transferred to the highestbidder = 220*100 = 22000$ 

Token amount that should be transferred = 220$.

Here the tokens is directly transferred before converting them into the terms of `convertibleBaseAsset` which causes the main issue.

### Recommendation

Before transferring the amount to the highest bidder, ping the oracle for the correct price of `convertibleBaseAsset` and then convert the riskFundBidAmount in the terms of `convertibleBaseAsset` Tokens. Even if a token pegged to USD is used, the oracle should be used to get the correct value AND the tokens should always be converted in terms of  `convertibleBaseAsset` as sometimes the pegged tokens might also divert from their price or decimals might be different for different tokens.


## Assessed type

Token-Transfer