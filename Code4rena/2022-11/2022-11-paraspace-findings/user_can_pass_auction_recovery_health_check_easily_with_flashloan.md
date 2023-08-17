## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- H-07

# [User can pass auction recovery health check easily with flashloan](https://github.com/code-423n4/2022-11-paraspace-findings/issues/478) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/pool/PoolParameters.sol#L281


# Vulnerability details

## Description

ParaSpace features an auction mechanism to liquidate user's NFT holdings and receive fair value. User has the option, before liquidation actually happens but after auction started, to top up their account to above recovery factor (> 1.5 instead of > 1) and use setAuctionValidityTime() to invalidate the auction.
```
require(
    erc721HealthFactor > ps._auctionRecoveryHealthFactor,
    Errors.ERC721_HEALTH_FACTOR_NOT_ABOVE_THRESHOLD
);
userConfig.auctionValidityTime = block.timestamp;
```

The issue is that the check validates the account is topped in the moment the TX is executed. Therefore, user may very easily make it appear they have fully recovered by borrowing a large amount of funds, depositing them as collateral, registering auction invalidation, removing the collateral and repaying the flash loan. Reentrancy guards are not effective to prevent this attack because all these actions are done in a sequence, one finishes before the other begins. However, it is clear user cannot immediately finish this attack below liquidation threshold because health factor check will not allow it.

Still, the recovery feature is a very important feature of the protocol and a large part of what makes it unique, which is why I think it is very significant that it can be bypassed.
I am on the fence on whether this should be HIGH or MED level impact, would support judge's verdict either way.

## Impact

User can pass auction recovery health check easily with flashloan

## Proof of Concept

1. User places NFT as collateral in the protocol
2. User borrows using the NFT as collateral
3. NFT price drops and health factor is lower than liquidation threshold
4. Auction to sell NFT initiates
5. User deposits just enough to be above liquidation threshold
6. User now flashloans 1000 WETH
	1. supply 1000 WETH to the protocol
	2. call setAuctionValidityTime(), cancelling the auction
	3. withdraw the 1000 WETH from the protocol
	4. pay back the 1000 WETH flashloan
7. End result is bypassing of recovery health check

## Tools Used

Manual audit

## Recommended Mitigation Steps

In order to know user has definitely recovered, implement it as a function which holds the user's assets for X time (at least 5 minutes), then releases it back to the user and cancelling all their auctions.