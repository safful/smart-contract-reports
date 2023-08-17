## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-01

# [It is possible to steal the unallocated part of every delegation period budget](https://github.com/code-423n4/2023-05-ajna-findings/issues/465) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-grants/src/grants/base/StandardFunding.sol#L316-L318


# Vulnerability details

Attacker can monitor the standard proposals distribution and routinely steal each low activity period remainder by submitting a `transfer to self` proposal and voting a dust amount for it.

Since the criteria for the final slate update is that any increase in total funding votes casted is enough, the attacker's costs are negligible, while the remainder funds during some periods can be substantial enough for the attacker to setup such a monitoring. I.e. as funds are constant share of the treasury, while activity can differ drastically, a situation when there are less viable proposals then funds can routinely happen over time.

The assumption of the current logic is that such unallocated funds will be returned to the treasury, but it will not be the case as the cost of stealing such funds is close to zero.

## Impact

A part of treasury funds can be stolen each period and will not be available for ecosystem funding.

## Proof of Concept

Schematic POC:

1. Bob monitors the end of each screening period and, whenever it is cheap enough, submits a proposal to send the remainder funds to self via proposeStandard()
2. Bob votes for it with fundingVote() with the dust votes he have. Since it is low activity period there are room, and it is included to `_topTenProposals`
3. Bob updates the top slate with updateSlate(), repeating current top slate with his proposal added. Since other proposals cumulatively do not allocate full budget and Bob's proposal have positive funding vote attached, it is included to the slate

This way Bob obtained the remainder funds nearly for free.

Core issue here looks to be the absence of the proposal votes threshold, which allows an attacker to claim the remained without any barrier to entry, i.e. having at hand only dust amount of governance tokens.

Even proposal with zero funding votes can be executed, it is only controlled to be non-negative:

https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-grants/src/grants/base/StandardFunding.sol#L421-L441

```solidity
    function _validateSlate(uint24 distributionId_, uint256 endBlock, uint256 distributionPeriodFundsAvailable_, uint256[] calldata proposalIds_, uint256 numProposalsInSlate_) internal view returns (uint256 sum_) {
        // check that the function is being called within the challenge period
        if (block.number <= endBlock || block.number > _getChallengeStageEndBlock(endBlock)) {
            revert InvalidProposalSlate();
        }

        // check that the slate has no duplicates
        if (_hasDuplicates(proposalIds_)) revert InvalidProposalSlate();

        uint256 gbc = distributionPeriodFundsAvailable_;
        uint256 totalTokensRequested = 0;

        // check each proposal in the slate is valid
        for (uint i = 0; i < numProposalsInSlate_; ) {
            Proposal memory proposal = _standardFundingProposals[proposalIds_[i]];

            // check if Proposal is in the topTenProposals list
            if (_findProposalIndex(proposalIds_[i], _topTenProposals[distributionId_]) == -1) revert InvalidProposalSlate();

            // account for fundingVotesReceived possibly being negative
>>          if (proposal.fundingVotesReceived < 0) revert InvalidProposalSlate();
```

The only criteria for state update is greater sum of the funding votes:

https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-grants/src/grants/base/StandardFunding.sol#L316-L318

```solidity
        // check if slate of proposals is better than the existing slate, and is thus the new top slate
        newTopSlate_ = currentSlateHash == 0 ||
>>          (currentSlateHash!= 0 && sum > _sumProposalFundingVotes(_fundedProposalSlates[currentSlateHash]));
```

I.e. when the activity is low enough attacker can always maximize the `totalTokensRequested` to be exactly `gbc * 9 / 10`, claiming the difference to itself (i.e. the dust vote supplied proposal is to transfer unallocated part to attacker's account in this case):

https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-grants/src/grants/base/StandardFunding.sol#L445-L450

```solidity
            totalTokensRequested += proposal.tokensRequested;

            // check if slate of proposals exceeded budget constraint ( 90% of GBC )
>>          if (totalTokensRequested > (gbc * 9 / 10)) {
                revert InvalidProposalSlate();
            }
```

## Recommended Mitigation Steps

Consider introducing the minimum accepted vote power for any proposal to be included in the final slate, as an example:

https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-grants/src/grants/base/StandardFunding.sol#L421-L441

```diff
    function _validateSlate(uint24 distributionId_, uint256 endBlock, uint256 distributionPeriodFundsAvailable_, uint256[] calldata proposalIds_, uint256 numProposalsInSlate_) internal view returns (uint256 sum_) {
        ...

+       // using 0.1% of the total vote power used as a minimum for any winning proposal
+       uint minFundingVotePower = _distributions[distributionId_].fundingVotePowerCast / 1000;
        // check each proposal in the slate is valid
        for (uint i = 0; i < numProposalsInSlate_; ) {
            Proposal memory proposal = _standardFundingProposals[proposalIds_[i]];

            // check if Proposal is in the topTenProposals list
            if (_findProposalIndex(proposalIds_[i], _topTenProposals[distributionId_]) == -1) revert InvalidProposalSlate();

            // account for fundingVotesReceived possibly being negative
-           if (proposal.fundingVotesReceived < 0) revert InvalidProposalSlate();
+           if (proposal.fundingVotesReceived < 0 || Maths.wpow(SafeCast.toUint256(Maths.abs(proposal.fundingVotesReceived)), 2) < minFundingVotePower) revert InvalidProposalSlate();
```


## Assessed type

Invalid Validation