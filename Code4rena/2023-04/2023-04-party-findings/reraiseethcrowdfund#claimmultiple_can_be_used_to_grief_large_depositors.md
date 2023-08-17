## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-04

# [ReraiseETHCrowdfund#claimMultiple can be used to grief large depositors](https://github.com/code-423n4/2023-04-party-findings/issues/35) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ReraiseETHCrowdfund.sol#L333-L382


# Vulnerability details

## Impact

User can be grieved by being force minted a large number of NFTs with low voting power instead of one with high voting power

## Proof of Concept

[ReraiseETHCrowdfund.sol#L354-L377](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ReraiseETHCrowdfund.sol#L354-L377)

        for (uint256 i; i < votingPowerByCard.length; ++i) {
            if (votingPowerByCard[i] == 0) continue;

            uint96 contribution = (votingPowerByCard[i] * 1e4) / exchangeRateBps;
            if (contribution < minContribution_) {
                revert BelowMinimumContributionsError(contribution, minContribution_);
            }

            if (contribution > maxContribution_) {
                revert AboveMaximumContributionsError(contribution, maxContribution_);
            }

            votingPower -= votingPowerByCard[i];

            // Mint contributor a new party card.
            uint256 tokenId = party.mint(contributor, votingPowerByCard[i], delegate);

            emit Claimed(contributor, tokenId, votingPowerByCard[i]);
        }

ReraiseETHCrowdfund#claimMultiple can be called by any user for any other user. The above loop uses the user specified `votingPowerByCard` to assign each token a voting power and mint them to the contributor. This is problematic because large contributors can have their voting power fragmented into a large number of NFTs which a small amount of voting power each. The dramatically inflates the gas costs of the affected user.

Example:
minContribution = 1 and maxContribution = 100. User A contributes 100. This means they should qualify for one NFT of the largest size. However instead they can be minted 100 NFTs with 1 vote each. 

## Tools Used

Manual Review

## Recommended Mitigation Steps

If msg.sender isn't contributor it should force the user to mint the minimum possible number of NFTs:

        uint256 votingPower = pendingVotingPower[contributor];

        if (votingPower == 0) return;

    +  if (msg.sender != contributor) {
    +      require(votingPowerByCard.length == (((votingPower - 1)/maxContribution) + 1));
    +  }