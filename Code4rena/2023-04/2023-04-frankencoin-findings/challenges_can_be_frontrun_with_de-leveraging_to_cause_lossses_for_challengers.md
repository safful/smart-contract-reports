## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-01

# [Challenges can be frontrun with de-leveraging to cause lossses for challengers](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/945) 

# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/MintingHub.sol#L140-L148


# Vulnerability details

## Impact
Challenges, once created, cannot be closed. Thus once a challenge is created, the challenger has already transferred in a collateral amount and is thus open for losing their collateral to a bidding war which will most likely close below market price, since otherwise buying from the market would be cheaper for bidders.

Position owners can take advantage of this fact and frontrun a `launchChallenge` transaction with an `adjustPrice` transaction. The `adjustPrice` function lets the user lower the price of the position, and can pass the collateral check by sending collateral tokens externally.

As a worst case scenario, consider a case where a position is open with 1 ETH collateral and 1500 ZCHF minted. A challenger challenges the position and the owner frontruns the challenger by sending the contract 1500 ZCHF and calling `repay()` and then calling `adjustPrice` with value 0, all in one transaction with a contract. Now, the price in the contract is set to 0, and the collateral check passes since the outstanding minted amount is 0. The challenger's transaction gets included next, and they are now bidding away their collateral, since any amount of bid will pass the avert collateral check.

The position owner themselves can backrun the same transaction with a bid of 1 wei and take all the challenger's collateral, since every bid checks for the `tryAvertChallenge` condition.
```solidity
if (_bidAmountZCHF * ONE_DEC18 >= price * _collateralAmount)
```

Since price is set to 0, any bid passes this check. This sandwich attack causes immense losses to all challengers in the system, baiting them with bad positions and then sandwiching their challenges.

Since sandwich attacks are extremely commonplace, this is classified as high severity.
## Proof of Concept
The attack can be performed the following steps.

1. Have an undercollateralized position. This can be caused naturally due to market movements.
2. Frontrun challenger's transaction with a repayment and `adjustPrice` call lowering the price.
3. Challenger's call gets included, where they now put up collateral for bids.
4. Backrun challenger's call with a bid such that it triggers the avert.
5. Attacker just claimed the challenger's collateral at their specified bid price, which can be as little as 1 wei if price is 0.

## Tools Used
Manual Review
## Recommended Mitigation Steps
When launching a challenge, ask for a `expectedPrice` argument. If the actual price does not match this expected price, that means that transaction was frontrun and should be reverted. This acts like a slippage check for challenges.