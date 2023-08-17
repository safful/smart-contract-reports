## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- H-02

# [An attacker can contribute to the ETH crowdfund using a flash loan and control the party as he likes.](https://github.com/code-423n4/2023-04-party-findings/issues/25) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L273
https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernance.sol#L470


# Vulnerability details

## Impact
An attacker can have more than half of the total voting power using a flash loan and abuse other contributors.

## Proof of Concept
The main flaw is that the party can distribute funds right after the crowdfund is finalized within the same block.

So the attacker can contribute using a flash loan and repay by distributing the part's ETH.

1. Let's assume `maxContribution = type(uint96).max, minTotalContributions = 10 ether, maxTotalContributions = 20 ether, fundingSplitBps = 0`.
2. An attacker contributes 1 ether(attacker's fund) to the crowdfund and another user contributes 9 ether.
3. The attacker knows the crowdfund will be finalized as it satisfies the `minTotalContributions` already but he will have 10% of the total voting power.
4. So he decides to contribute 10 ether using a flash loan.
5. In `ETHCrowdfundBase._processContribution()`, the crowdfund will be finalized immediately as [total contribution is greater than maxTotalContributions](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L212).
6. Then the attacker will have `(1 + 10) / 20 = 55%` voting power of the party and he can pass any proposal.
7. So he calls `distribute()` with 19 ether. `distribute()` can be called directly if `opts.distributionsRequireVote == false`, otherwise, he should create/execute the distribution proposal and he can do it within the same block.
8. After that, he can receive ETH using `TokenDistributor.claim()` and the amount will be `19 * 55% = 10.45 ether`. (We ignore the distribution fee for simplicity)
9. He repays 10 ether to the flash loan provider and he can control the party as he likes now.

This attack is possible for both `InitialETHCrowdfund` and `ReraiseETHCrowdfund`.

## Tools Used
Manual Review

## Recommended Mitigation Steps
I think we should implement a kind of `cooldown logic` after the crowdfund is finalized.

1. Add a `partyStartedTime` in `PartyGovernance.sol`.
2. While finalizing the ETH crowdfund in `ETHCrowdfundBase._finalize()`, we set `party.partyStartedTime = block.timestamp`.
3. After that, `PartyGovernance.distribute()` can work only when `block.timestamp > partyStartTime`.