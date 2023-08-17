## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- selected for report
- sponsor acknowledged
- edited-by-warden
- M-13

# [no bidder has incentive to bid the Auction except doing last-minute MEV due to fixed endBlock](https://github.com/code-423n4/2023-06-stader-findings/issues/70) 

# Lines of code

https://github.com/code-423n4/2023-06-stader/blob/main/contracts/Auction.sol#L48-L50


# Vulnerability details

## Impact
no bidder has incentive to bid the Auction except doing last-minute MEV due to fixed endBlock

## Proof of Concept
The auction of SD Token has a fixed endBlock. bidder(s) would like to get SD Token with least ETH and they are all incentivized to just bid at the last block, leading to loss of protocol principle (during the auction).


```solidity
    function createLot(uint256 _sdAmount) external override whenNotPaused {
        lots[nextLot].startBlock = block.number;
        lots[nextLot].endBlock = block.number + duration;
        lots[nextLot].sdAmount = _sdAmount;

```

Genearally, auction with fixed endtime has the known vulnerability of being bidden at the last block, essentially the validator/MEVer who has the ability to slip in transaction at that block has the highest likelihood to get the bid. This basically give them advantage, and would lead to the auction to end at lower price.

## Tools Used

## Recommended Mitigation Steps
Extend the final endBlock at each bid. This can be activated at the end of 1h for example to ensure the highest bidder can take the auction on a fairly manner.






## Assessed type

MEV