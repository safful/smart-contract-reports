## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- M-07

# [Calling `StandardFunding.screeningVote` function and `ExtraordinaryFunding.voteExtraordinary` function when `block.number` equals respective start block and when `block.number` is bigger than respective start block can result in different available votes for same voter](https://github.com/code-423n4/2023-05-ajna-findings/issues/288) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/StandardFunding.sol#L519-L569
https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/StandardFunding.sol#L891-L910
https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/Funding.sol#L76-L93
https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/StandardFunding.sol#L572-L596
https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/StandardFunding.sol#L698-L753
https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/StandardFunding.sol#L872-L881
https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L131-L157
https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L246-L256


# Vulnerability details

## Impact
Because of `if (block.number <= screeningStageEndBlock || block.number > endBlock) revert InvalidVote()`, the following `StandardFunding.fundingVote` function can only execute `uint128 newVotingPower = SafeCast.toUint128(_getVotesFunding(msg.sender, votingPower, voter.remainingVotingPower, screeningStageEndBlock))` when `block.number` is bigger than `screeningStageEndBlock`.

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/StandardFunding.sol#L519-L569
```solidity
    function fundingVote(
        FundingVoteParams[] memory voteParams_
    ) external override returns (uint256 votesCast_) {
        uint24 currentDistributionId = _currentDistributionId;

        QuarterlyDistribution storage currentDistribution = _distributions[currentDistributionId];
        QuadraticVoter        storage voter               = _quadraticVoters[currentDistributionId][msg.sender];

        uint256 endBlock = currentDistribution.endBlock;

        uint256 screeningStageEndBlock = _getScreeningStageEndBlock(endBlock);

        // check that the funding stage is active
        if (block.number <= screeningStageEndBlock || block.number > endBlock) revert InvalidVote();

        uint128 votingPower = voter.votingPower;

        // if this is the first time a voter has attempted to vote this period,
        // set initial voting power and remaining voting power
        if (votingPower == 0) {

            // calculate the voting power available to the voting power in this funding stage
            uint128 newVotingPower = SafeCast.toUint128(_getVotesFunding(msg.sender, votingPower, voter.remainingVotingPower, screeningStageEndBlock));

            voter.votingPower          = newVotingPower;
            voter.remainingVotingPower = newVotingPower;
        }

        ...
    }
```

When the `StandardFunding.fundingVote` function calls the following `StandardFunding._getVotesFunding` function, `screeningStageEndBlock` would be used as the `voteStartBlock_` input for calling the `Funding._getVotesAtSnapshotBlocks` function below. Because `block.number` would always be bigger than `screeningStageEndBlock`, `voteStartBlock_` would always be `screeningStageEndBlock` in the `Funding._getVotesAtSnapshotBlocks` function. This means that the `Funding._getVotesAtSnapshotBlocks` function would always return the same voting power for the same voter at any `block.number` that is bigger than `screeningStageEndBlock` during the funding phase.

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/StandardFunding.sol#L891-L910
```solidity
    function _getVotesFunding(
        address account_,
        uint256 votingPower_,
        uint256 remainingVotingPower_,
        uint256 screeningStageEndBlock_
    ) internal view returns (uint256 votes_) {
        // voter has already allocated some of their budget this period
        if (votingPower_ != 0) {
            votes_ = remainingVotingPower_;
        }
        // voter hasn't yet called _castVote in this period
        else {
            votes_ = Maths.wpow(
            _getVotesAtSnapshotBlocks(
                account_,
                screeningStageEndBlock_ - VOTING_POWER_SNAPSHOT_DELAY,
                screeningStageEndBlock_
            ), 2);
        }
    }
```

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/Funding.sol#L76-L93
```solidity
    function _getVotesAtSnapshotBlocks(
        address account_,
        uint256 snapshot_,
        uint256 voteStartBlock_
    ) internal view returns (uint256) {
        IVotes token = IVotes(ajnaTokenAddress);

        // calculate the number of votes available at the snapshot block
        uint256 votes1 = token.getPastVotes(account_, snapshot_);

        // enable voting weight to be calculated during the voting period's start block
        voteStartBlock_ = voteStartBlock_ != block.number ? voteStartBlock_ : block.number - 1;

        // calculate the number of votes available at the stage's start block
        uint256 votes2 = token.getPastVotes(account_, voteStartBlock_);

        return Maths.min(votes2, votes1);
    }
```

However, because of `if (block.number < currentDistribution.startBlock || block.number > _getScreeningStageEndBlock(currentDistribution.endBlock)) revert InvalidVote()`, calling the following `StandardFunding.screeningVote` function would not revert when `block.number` equals the current distribution period's start block.

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/StandardFunding.sol#L572-L596
```solidity
    function screeningVote(
        ScreeningVoteParams[] memory voteParams_
    ) external override returns (uint256 votesCast_) {
        QuarterlyDistribution memory currentDistribution = _distributions[_currentDistributionId];

        // check screening stage is active
        if (block.number < currentDistribution.startBlock || block.number > _getScreeningStageEndBlock(currentDistribution.endBlock)) revert InvalidVote();

        uint256 numVotesCast = voteParams_.length;

        for (uint256 i = 0; i < numVotesCast; ) {
            Proposal storage proposal = _standardFundingProposals[voteParams_[i].proposalId];

            // check that the proposal is part of the current distribution period
            if (proposal.distributionId != currentDistribution.id) revert InvalidVote();

            uint256 votes = voteParams_[i].votes;

            // cast each successive vote
            votesCast_ += votes;
            _screeningVote(msg.sender, proposal, votes);

            unchecked { ++i; }
        }
    }
```

When the `StandardFunding.screeningVote` function calls the following `StandardFunding._screeningVote` function, `if (screeningVotesCast[distributionId][account_] + votes_ > _getVotesScreening(distributionId, account_)) revert InsufficientVotingPower()` is executed in which the `StandardFunding._getVotesScreening` function below would use the current distribution period's start block as the `voteStartBlock_` input for calling the `Funding._getVotesAtSnapshotBlocks` function. Since the `Funding._getVotesAtSnapshotBlocks` function executes `voteStartBlock_ = voteStartBlock_ != block.number ? voteStartBlock_ : block.number - 1`, `voteStartBlock_` would be 1 block prior to the current distribution period's start block when `block.number` equals the current distribution period's start block, and `voteStartBlock_` would be the current distribution period's start block when `block.number` is bigger than the current distribution period's start block. However, it is possible that the same voter has different available votes at 1 block prior to the current distribution period's start block and at the current distribution period's start block. This is unlike the `StandardFunding._getVotesFunding` function that would always return the same voting power for the same voter when calling the `StandardFunding.fundingVote` function during the funding phase. Since calling the `StandardFunding._getVotesScreening` function when `block.number` equals the current distribution period's start block and when `block.number` is bigger than the current distribution period's start block during the screening phase can return different available votes for the same voter, this voter would call the `StandardFunding.screeningVote` function at a chosen `block.number` that would provide the highest votes.

This should not be allowed because `_getVotesScreening(distributionId, account_)` needs to return the same number of votes across all blocks during the screening phase to make `if (screeningVotesCast[distributionId][account_] + votes_ > _getVotesScreening(distributionId, account_)) revert InsufficientVotingPower()` effective in the `StandardFunding._screeningVote` function. For example, a voter who has no available votes at 1 block prior to the current distribution period's start block can mint many AJNA tokens at the current distribution period's start block and call the `StandardFunding.screeningVote` function at `block.number` that is bigger than the current distribution period's start block during the screening phase to use her or his available votes at current distribution period's start block. For another example, a voter who has available votes at 1 block prior to the current distribution period's start block can call the `StandardFunding.screeningVote` function when `block.number` equals the current distribution period's start block and then sell all of her or his AJNA tokens at the same `block.number`. Such voters' actions are unfair to other voters who maintain the same number of available votes at 1 block prior to the current distribution period's start block and at the current distribution period's start block for the screening stage voting.

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/StandardFunding.sol#L698-L753
```solidity
    function _screeningVote(
        address account_,
        Proposal storage proposal_,
        uint256 votes_
    ) internal {
        uint24 distributionId = proposal_.distributionId;

        // check that the voter has enough voting power to cast the vote
        if (screeningVotesCast[distributionId][account_] + votes_ > _getVotesScreening(distributionId, account_)) revert InsufficientVotingPower();

        ...
    }
```

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/StandardFunding.sol#L872-L881
```solidity
    function _getVotesScreening(uint24 distributionId_, address account_) internal view returns (uint256 votes_) {
        uint256 startBlock = _distributions[distributionId_].startBlock;

        // calculate voting weight based on the number of tokens held at the snapshot blocks of the screening stage
        votes_ = _getVotesAtSnapshotBlocks(
            account_,
            startBlock - VOTING_POWER_SNAPSHOT_DELAY,
            startBlock
        );
    }
```

Please note that calling the following `ExtraordinaryFunding.voteExtraordinary` function when `block.number` equals `_extraordinaryFundingProposals[proposalId_].startBlock` also does not revert, and the `ExtraordinaryFunding._getVotesExtraordinary` function below also uses `_extraordinaryFundingProposals[proposalId_].startBlock` as the `voteStartBlock_` input for calling the `Funding._getVotesAtSnapshotBlocks` function. Hence, the same issue also applies to the `ExtraordinaryFunding.voteExtraordinary` function.

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L131-L157
```solidity
    function voteExtraordinary(
        uint256 proposalId_
    ) external override returns (uint256 votesCast_) {
        // revert if msg.sender already voted on proposal
        if (hasVotedExtraordinary[proposalId_][msg.sender]) revert AlreadyVoted();

        ExtraordinaryFundingProposal storage proposal = _extraordinaryFundingProposals[proposalId_];
        // revert if proposal is inactive
        if (proposal.startBlock > block.number || proposal.endBlock < block.number || proposal.executed) {
            revert ExtraordinaryFundingProposalInactive();
        }

        // check voting power at snapshot block and update proposal votes
        votesCast_ = _getVotesExtraordinary(msg.sender, proposalId_);
        proposal.votesReceived += SafeCast.toUint120(votesCast_);

        // record that voter has voted on this extraordinary funding proposal
        hasVotedExtraordinary[proposalId_][msg.sender] = true;

        ...
    }
```

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L246-L256
```solidity
    function _getVotesExtraordinary(address account_, uint256 proposalId_) internal view returns (uint256 votes_) {
        if (proposalId_ == 0) revert ExtraordinaryFundingProposalInactive();

        uint256 startBlock = _extraordinaryFundingProposals[proposalId_].startBlock;

        votes_ = _getVotesAtSnapshotBlocks(
            account_,
            startBlock - VOTING_POWER_SNAPSHOT_DELAY,
            startBlock
        );
    }
```

## Proof of Concept
The following steps can occur for the described scenario.
1. At 1 block prior to the current distribution period's start block, Alice has no available votes at all.
2. After noticing the `StandardFunding.startNewDistributionPeriod` transaction that would end the previous distribution period and starts the current distribution period in the mempool, Alice backruns that transaction by minting a lot of AJNA tokens at the current distribution period's start block.
3. When `block.number` becomes bigger than the current distribution period's start block during the screening phase, Alice calls the `StandardFunding.screeningVote` function to successfully use all of her available votes at the current distribution period's start block for a proposal.
4. Alice's actions are unfair to Bob who prepares for the screening stage voting and maintains the same number of available votes at 1 block prior to the current distribution period's start block and at the current distribution period's start block.

## Tools Used
VSCode

## Recommended Mitigation Steps
https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/StandardFunding.sol#L578 can be updated to the following code:
```solidity
        if (block.number <= currentDistribution.startBlock || block.number > _getScreeningStageEndBlock(currentDistribution.endBlock)) revert InvalidVote();
```

and

https://github.com/code-423n4/2023-05-ajna/blob/main/ajna-grants/src/grants/base/ExtraordinaryFunding.sol#L139-L141 can be updated to the following code:
```solidity
        if (proposal.startBlock >= block.number || proposal.endBlock < block.number || proposal.executed) {
            revert ExtraordinaryFundingProposalInactive();
        }
```


## Assessed type

Timing