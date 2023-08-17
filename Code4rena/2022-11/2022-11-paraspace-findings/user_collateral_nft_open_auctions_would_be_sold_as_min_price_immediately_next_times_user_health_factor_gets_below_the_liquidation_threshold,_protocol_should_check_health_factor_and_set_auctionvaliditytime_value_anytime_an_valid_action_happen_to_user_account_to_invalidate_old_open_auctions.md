## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- M-09

# [user collateral NFT open auctions would be sold as min price immediately next times user health factor gets below the liquidation threshold, protocol should check health factor and set auctionValidityTime value anytime an valid action happen to user account to invalidate old open auctions](https://github.com/code-423n4/2022-11-paraspace-findings/issues/323) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/pool/DefaultReserveAuctionStrategy.sol#L90-L135
https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/tokenization/libraries/MintableERC721Logic.sol#L424-L434


# Vulnerability details

## Impact
attacker can liquidate users NFT collaterals with min price immediately after user health factor gets below the liquidation threshold for NFT collaterals that have old open auctions and `setAuctionValidityTime()` is not get called for the user. whenever user's account health factor gets below the NFT liquidation threshold attacker can start auction for all of users NFTs in the protocol. user need to call `setAuctionValidityTime()` to invalidated all of those open auctions whenever his/her account health factor is good, but if user don't call this and don't close those auctions then next time user's accounts health factor gets below the threshold attacker would liquidate all of those NFTs with minimum price immediately.

## Proof of Concept
This is `calculateAuctionPriceMultiplier()` code in DefaultReserveAuctionStrategy:
```
    function calculateAuctionPriceMultiplier(
        uint256 auctionStartTimestamp,
        uint256 currentTimestamp
    ) external view override returns (uint256) {
        uint256 ticks = PRBMathUD60x18.div(
            currentTimestamp - auctionStartTimestamp,
            _tickLength
        );
        return _calculateAuctionPriceMultiplierByTicks(ticks);
    }

    function _calculateAuctionPriceMultiplierByTicks(uint256 ticks)
        internal
        view
        returns (uint256)
    {
        if (ticks < PRBMath.SCALE) {
            return _maxPriceMultiplier;
        }

        uint256 ticksMinExp = PRBMathUD60x18.div(
            (PRBMathUD60x18.ln(_maxPriceMultiplier) -
                PRBMathUD60x18.ln(_minExpPriceMultiplier)),
            _stepExp
        );
        if (ticks <= ticksMinExp) {
            return
                PRBMathUD60x18.div(
                    _maxPriceMultiplier,
                    PRBMathUD60x18.exp(_stepExp.mul(ticks))
                );
        }

        uint256 priceMinExpEffective = PRBMathUD60x18.div(
            _maxPriceMultiplier,
            PRBMathUD60x18.exp(_stepExp.mul(ticksMinExp))
        );
        uint256 ticksMin = ticksMinExp +
            (priceMinExpEffective - _minPriceMultiplier).div(_stepLinear);

        if (ticks <= ticksMin) {
            return priceMinExpEffective - _stepLinear.mul(ticks - ticksMinExp);
        }

        return _minPriceMultiplier;
    }
```
As you can see when long times passed from auction start time the price of auction would be minimum.
This is `isAuctioned()` code in MintableERC721Data contract:
```

    function isAuctioned(
        MintableERC721Data storage erc721Data,
        IPool POOL,
        uint256 tokenId
    ) public view returns (bool) {
        return
            erc721Data.auctions[tokenId].startTime >
            POOL
                .getUserConfiguration(erc721Data.owners[tokenId])
                .auctionValidityTime;
    }
```
As you can see auction is valid if startTime is bigger than user's `auctionValidityTime`. but the value of `auctionValidityTime` should be set manually by calling `PoolParameters.setAuctionValidityTime()`, so by default auction would stay open. imagine this scenario:
1. user NFT health factor gets under the liquidation threshold.
2. attacker calls `startAuction()` for all user collateral NFT tokens.
3. user supply some collateral and his health factor become good.
4. some time passes and user NFT health factor gets under the liquidation threshold.
5. attacker can liquidate all of user NFT collateral with minimum price of auction immediately.
6. user would lose his NFTs without proper auction.

This scenario can be common because liquidation and health factor changes happens on-chain and most of the user isn't always watches his account health factor and he wouldn't know when his account health factor gets below the liquidation threshold and when it's needed for him to call `setAuctionValidityTime()`. so over time this would happen to more users.

## Tools Used
VIM

## Recommended Mitigation Steps
contract should check and set the value of `auctionValidityTime` for user whenever an action happens to user account.
also there should be some incentive mechanism for anyone starting or ending an auction.