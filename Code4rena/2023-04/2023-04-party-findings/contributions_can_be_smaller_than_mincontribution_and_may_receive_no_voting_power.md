## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- M-03

# [Contributions can be smaller than minContribution and may receive no voting power](https://github.com/code-423n4/2023-04-party-findings/issues/37) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L169-L234


# Vulnerability details

## Impact

Valid contribution is awarded no voting power

## Proof of Concept

[ETHCrowdfundBase.sol#L195-L219](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L195-L219)

        uint96 minContribution_ = minContribution;
        if (amount < minContribution_) {
            revert BelowMinimumContributionsError(amount, minContribution_);
        }
        uint96 maxContribution_ = maxContribution;
        if (amount > maxContribution_) {
            revert AboveMaximumContributionsError(amount, maxContribution_);
        }

        uint96 newTotalContributions = totalContributions + amount;
        uint96 maxTotalContributions_ = maxTotalContributions;
        if (newTotalContributions >= maxTotalContributions_) {
            totalContributions = maxTotalContributions_;

            // Finalize the crowdfund.
            // This occurs before refunding excess contribution to act as a
            // reentrancy guard.
            _finalize(maxTotalContributions_);

            // Refund excess contribution.
            uint96 refundAmount = newTotalContributions - maxTotalContributions;
            if (refundAmount > 0) {
                amount -= refundAmount; <- @audit-issue amount is reduced after min check
                payable(msg.sender).transferEth(refundAmount);
            }

When processing a contribution, if the amount contributed would push the crowdfund over the max then it is reduced. This is problematic because this reduction occurs AFTER it checks the amount against the minimum contribution. The result is that these contributions can end up being less than the specified minimum.

Although an edge case, if amount is smaller than exchangeRateBps as it could result in the user receiving no voting power at all for their contribution.

## Tools Used

Manual Review

## Recommended Mitigation Steps

Enforce minContribution after reductions to amount