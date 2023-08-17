## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-11

# [Later challengers can bid on the previous challenge to extend the expiration time of the previous challenge, so that their own challenge can succeed before the previous challenge and get challenge rewards](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/349) 

# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/MintingHub.sol#L217-L224


# Vulnerability details

## Impact
When bidders bid, if the expiration time of the challenge is less than 30 minutes, the expiration time will be extended.
```solidity
            uint256 earliestEnd = block.timestamp + 30 minutes;
            if (earliestEnd >= challenge.end) {
                // bump remaining time like ebay does when last minute bids come in
                // An attacker trying to postpone the challenge forever must increase the bid by 0.5%
                // every 30 minutes, or double it every three days, making the attack hard to sustain
                // for a prolonged period of time.
                challenge.end = earliestEnd;
            }
```
However, extending the expiration time will break the order of the challenges, so that the later challenges will succeed before the previous ones, thus affecting the challenger's reward expectations.

Consider the following scenario
There is a collateral of 2 WETH in a position, and as the actual price of WETH drops, challengers are attracted to challenge it.
In block 1, alice uses 2 WETH to challenge the position, and the expiration time is block 7201
At block 2, bob challenges the position with 2 WETH, expiring at block 7202
The bidder then bids 4000 ZCHF each for alice's and bob's challenges.
In block 7200, bob finds that if alice's challenge is successful, then bob will not be able to get the challenge reward, so bob bids 4200 ZCHF to alice's challenge. Alice's challenge expiration time is extended to block 7351.
At block 7201, alice cannot call end to make the challenge successful because the expiration time is extended
At block 7202, bob successfully calls end to make his challenge successful and gets the challenge reward.
In block 7351, alice calls the end function. Since the collateral in the position is 0 at this time, alice will not be able to get the challenge reward, and bob's 4200 zchf will be returned.
```solidity
    function notifyChallengeSucceeded(address _bidder, uint256 _bid, uint256 _size) external onlyHub returns (address, uint256, uint256, uint256, uint32) {
        challengedAmount -= _size;
        uint256 colBal = collateralBalance();
        if (_size > colBal){
            // Challenge is larger than the position. This can for example happen if there are multiple concurrent
            // challenges that exceed the collateral balance in size. In this case, we need to redimension the bid and
            // tell the caller that a part of the bid needs to be returned to the bidder.
            _bid = _divD18(_mulD18(_bid, colBal), _size);
            _size = colBal;
        }

        // Note that thanks to the collateral invariant, we know that
        //    colBal * price >= minted * ONE_DEC18
        // and that therefore
        //    price >= minted / colbal * E18
        // such that
        //    volumeZCHF = price * size / E18 >= minted * size / colbal
        // So the owner cannot maliciously decrease the price to make volume fall below the proportionate repayment.
        uint256 volumeZCHF = _mulD18(price, _size); // How much could have minted with the challenged amount of the collateral
        // The owner does not have to repay (and burn) more than the owner actually minted.  
        uint256 repayment = minted < volumeZCHF ? minted : volumeZCHF; // how much must be burned to make things even

```
## Proof of Concept
https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/MintingHub.sol#L217-L224
https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/Position.sol#L329-L350
## Tools Used
None
## Recommended Mitigation Steps
Consider implementing a challenge queue that allows the end function to be called on subsequent challenges only after previous challenges have ended