## Tags

- bug
- 3 (High Risk)
- satisfactory
- selected for report
- sponsor confirmed
- H-01

# [Self-delgated users can have their delegation unknowingly hijacked during crowdfunding](https://github.com/code-423n4/2023-04-party-findings/issues/38) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L169-L234


# Vulnerability details

## Impact

Self-delegation can be hijacked

## Proof of Concept

[PartyGovernance.sol#L886-L906](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernance.sol#L886-L906)

    function _adjustVotingPower(address voter, int192 votingPower, address delegate) internal {
        VotingPowerSnapshot memory oldSnap = _getLastVotingPowerSnapshotForVoter(voter);
        address oldDelegate = delegationsByVoter[voter];
        // If `oldDelegate` is zero and `voter` never delegated, then have
        // `voter` delegate to themself.
        oldDelegate = oldDelegate == address(0) ? voter : oldDelegate;
        // If the new `delegate` is zero, use the current (old) delegate.
        delegate = delegate == address(0) ? oldDelegate : delegate;

        VotingPowerSnapshot memory newSnap = VotingPowerSnapshot({
            timestamp: uint40(block.timestamp),
            delegatedVotingPower: oldSnap.delegatedVotingPower,
            intrinsicVotingPower: (oldSnap.intrinsicVotingPower.safeCastUint96ToInt192() +
                votingPower).safeCastInt192ToUint96(),
            isDelegated: delegate != voter
        });
        _insertVotingPowerSnapshot(voter, newSnap);
        delegationsByVoter[voter] = delegate;
        // Handle rebalancing delegates.
        _rebalanceDelegates(voter, oldDelegate, delegate, oldSnap, newSnap);
    }

Self-delegation is triggered when a user specifies their delegate as address(0). This means that if a user wishes to self-delegate they will can contribute to a crowdfund with delegate == address(0). 

[ETHCrowdfundBase.sol#L169-L181](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/crowdfund/ETHCrowdfundBase.sol#L169-L181)

    function _processContribution(
        address payable contributor,
        address delegate,
        uint96 amount
    ) internal returns (uint96 votingPower) {
        address oldDelegate = delegationsByContributor[contributor];
        if (msg.sender == contributor || oldDelegate == address(0)) {
            // Update delegate.
            delegationsByContributor[contributor] = delegate;
        } else {
            // Prevent changing another's delegate if already delegated.
            delegate = oldDelegate;
        }

This method of self-delegation is problematic when combined with _processContribution. When contributing for someone else, the caller is allowed to specify any delegate they wish. If that user is currently self delegated, then the newly specified delegate will overwrite their self delegation. This allows anyone to hijack the voting power of a self-delegated user. 

This can create serious issues for ReraiseETHCrowdfund because party NFTs are not minted until after the entire crowdfund is successful. Unlike InitialETHCrowdfund, this allows the attacker to hijack all of the user's newly minted votes.

Example:
minContribution = 1 and maxContribution = 100. User A contributes 100 to ReraiseETHCrowdfund. They wish to self-delegate so they call contribute with delegate == address(0). An attacker now contributes 1 on behalf of User A with themselves as the delegate. Now when the NFTs are claimed, they will be delegated to the attacker. 

## Tools Used

Manual Review

## Recommended Mitigation Steps

Self-delegation should be automatically hardcoded:

    +   if (msg.sender == contributor && delegate == address(0)) {
    +       delegationsByContributor[contributor] = contributor;
    +   }

        address oldDelegate = delegationsByContributor[contributor];
        if (msg.sender == contributor || oldDelegate == address(0)) {
            // Update delegate.
            delegationsByContributor[contributor] = delegate;
        } else {
            // Prevent changing another's delegate if already delegated.
            delegate = oldDelegate;
        }