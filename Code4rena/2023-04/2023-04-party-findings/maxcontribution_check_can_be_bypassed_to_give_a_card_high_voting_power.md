## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-02

# [MaxContribution check can be bypassed to give a card high voting power](https://github.com/code-423n4/2023-04-party-findings/issues/39) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/main/contracts/crowdfund/ReraiseETHCrowdfund.sol#L295-L298


# Vulnerability details

## Proof of Concept
ReraiseETHCrowdfund tries limit the voting power of each card by doing a min/maxContribution check in claim and claimMultiple.
```
            uint96 contribution = (votingPower * 1e4) / exchangeRateBps;
            uint96 maxContribution_ = maxContribution;
            // Check that the contribution equivalent of total pending voting
            // power is not above the max contribution range. This can happen
            // for contributors who contributed multiple times In this case, the
            // `claimMultiple` function should be called instead. This is done
            // so parties may use the minimum and maximum contribution values to
            // limit the voting power of each card (e.g.  a party desiring a "1
            // card = 1 vote"-like governance system where each card has equal
            // voting power).
            if (contribution > maxContribution_) {
                revert AboveMaximumContributionsError(contribution, maxContribution_);
            }
```
https://github.com/code-423n4/2023-04-party/blob/main/contracts/crowdfund/ReraiseETHCrowdfund.sol#L270-L282

```
            // Check that the contribution equivalent of voting power is within
            // contribution range. This is done so parties may use the minimum
            // and maximum contribution values to limit the voting power of each
            // card (e.g. a party desiring a "1 card = 1 vote"-like governance
            // system where each card has equal voting power).
            uint96 contribution = (votingPowerByCard[i] * 1e4) / exchangeRateBps;
            if (contribution < minContribution_) {
                revert BelowMinimumContributionsError(contribution, minContribution_);
            }

            if (contribution > maxContribution_) {
                revert AboveMaximumContributionsError(contribution, maxContribution_);
            }
```
https://github.com/code-423n4/2023-04-party/blob/main/contracts/crowdfund/ReraiseETHCrowdfund.sol#L357-L369


However, this check can be bypassed due to the following code segment
```
        else if (party.ownerOf(tokenId) == contributor) {
            // Increase voting power of contributor's existing party card.
            party.addVotingPower(tokenId, votingPower);
        } 
```
https://github.com/code-423n4/2023-04-party/blob/main/contracts/crowdfund/ReraiseETHCrowdfund.sol#L295-L298

Consider the following situation. Suppose ReraiseETHCrowdfund sets maximumContribution to only allow at most 3 units of voting power in each card. Some user X can contribute the maximum amount twice as 2 different contributor addresses A & B (both of which he controls). When the crowdfund has finalized, X can first call claim as A, then transfer the partyGovernanceNFT from A to B (note that while the crowdfundNFT can't be transferred, the partyGovernanceNFT can be transferred), and finally call claim as B to get a card with 6 units of voting power.

## Impact
The degree of impact really depends on the use case of the party. Some parties would like each card to represent a single vote - this would obviously violate that. Generally, it's not a great idea to allow a single card to hold a high amount of votes, so I'll leave this as a medium for now.

## Tools Used
Manual review

## Recommended Mitigation Steps
One solution is to restrict the maximum voting power on partyGovernanceNFT's side. It can check the votingPower of each card before [adding more votingPower](https://github.com/code-423n4/2023-04-party/blob/main/contracts/party/PartyGovernanceNFT.sol#L169) to it.
