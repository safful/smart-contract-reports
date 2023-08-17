## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-06

# [Challengers and bidders can collude together to restrict the minting of position owner](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/745) 

# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/Position.sol#L312


# Vulnerability details

## Proof of Concept

There is no restrictions as to how many challenges can occur at one given auction time. A challenger can potentially create an insanely large amount of challenges with a tiny amount of collateral for each challenge. 

```
    function launchChallenge(address _positionAddr, uint256 _collateralAmount) external validPos(_positionAddr) returns (uint256) {
        IPosition position = IPosition(_positionAddr);
        IERC20(position.collateral()).transferFrom(msg.sender, address(this), _collateralAmount);
        uint256 pos = challenges.length;
        challenges.push(Challenge(msg.sender, position, _collateralAmount, block.timestamp + position.challengePeriod(), address(0x0), 0));
        position.notifyChallengeStarted(_collateralAmount);
        emit ChallengeStarted(msg.sender, address(position), _collateralAmount, pos);
        return pos;
    }
```

Let's say if the fair price of 1000 ZCHF is 1 WETH, and the position owner sets his position at the fair price. Rationally, there will be no challenges and bidders because the price is fair. However, the challenger can attack the position owner this way:

1. Set a small amount of collateralAmount to challenge, ie 0.0001 WETH.
2. The bidder comes and bid a price over 1000* ZCHF (Not the actual amount*. The actual amount is an equivalent amount scaled to the collateral, but for simplicity sake let's just say 1000 ZCHF, ideally its like 1000 * 0.0001 worth)
3. Because the bid is higher than 1000* ZCHF, tryAvertChallenge() succeeds and the bidder buys the collateral from the challenger.
4. When tryAvertChallenge() succeeds, restrictMinting(1 days) is called to suspend the owner from minting for 1 additional day
5. If the challenger and the bidder is colluding or even the same person, then the bidder does not lose out because he is essentially buying the collateral for a higher price from himself. 
6. The challenger and bidder can repeat this attack and suspend the owner from minting. Such attack is possible because there is nothing much to lose other than a small amount of gas fees

```
    function tryAvertChallenge(uint256 _collateralAmount, uint256 _bidAmountZCHF) external onlyHub returns (bool) {
        if (block.timestamp >= expiration){
            return false; // position expired, let every challenge succeed
        } else if (_bidAmountZCHF * ONE_DEC18 >= price * _collateralAmount){
            // challenge averted, bid is high enough
            challengedAmount -= _collateralAmount;
            // Don't allow minter to close the position immediately so challenge can be repeated before
            // the owner has a chance to mint more on an undercollateralized position
//@audit-- calls restrictMinting if passed
            restrictMinting(1 days);
            return true;
        } else {
            return false;
        }
    }
```

```
    function restrictMinting(uint256 period) internal {
        uint256 horizon = block.timestamp + period;
        if (horizon > cooldown){
            cooldown = horizon;
        }
    }
```

## Impact

Minting for Position owner will be suspended for a long time.

## Tools Used

VSCode

## Recommended Mitigation Steps

In this Frankencoin protocol, the challenger never really loses. 

If the bid ends lower than the liquidation price, then the bidder wins because he bought the collateral at a lower market value. The challenger also wins because he gets his reward. The protocol owner loses because he sold his collateral at a lower market value. 

If the bid ends higher than the liquidation price, then the bidder loses because he bought the collateral at a higher market value. The challenger wins because he gets to trade his collateral for a higher market value. The protocol owner neither wins nor loses.

The particular POC above is one way a challenger can abuse his power to create many challenges without any sort of consequence in order to attack the owner. In the spirit of fairness, the challenger should also lose if he challenges wrongly. 

Every time a challenger issues a challenge, he should pay a small fix sum of money that will go to the owner if the bidder sets an amount higher than fair market value. (because that means that the protocol owner was right about the fair market value all along.) 

Although the position owner can be a bidder himself, if the position owner bids on his own position in order to win this small amount of money, the position owner will lose at the same time because he is buying the collateral at a higher-than-market price from the challenger, so this simultaneous gain and loss will balance out.