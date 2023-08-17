## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- edited-by-warden
- M-03

# [NounsDAOV3Proposals.cancel() should allow to cancel the proposal of the Expired state](https://github.com/code-423n4/2023-07-nounsdao-findings/issues/55) 

# Lines of code

https://github.com/nounsDAO/nouns-monorepo/blob/718211e063d511eeda1084710f6a682955e80dcb/packages/nouns-contracts/contracts/governance/NounsDAOV3Proposals.sol#L571-L581


# Vulnerability details

## Impact
cancel() does not allow to cancel proposals in the final states Canceled/Defeated/Expired/Executed/Vetoed.
```solidity
    function cancel(NounsDAOStorageV3.StorageV3 storage ds, uint256 proposalId) external {
        NounsDAOStorageV3.ProposalState proposalState = stateInternal(ds, proposalId);
        if (
            proposalState == NounsDAOStorageV3.ProposalState.Canceled ||
            proposalState == NounsDAOStorageV3.ProposalState.Defeated ||
            proposalState == NounsDAOStorageV3.ProposalState.Expired ||
            proposalState == NounsDAOStorageV3.ProposalState.Executed ||
            proposalState == NounsDAOStorageV3.ProposalState.Vetoed
        ) {
            revert CantCancelProposalAtFinalState();
        }
```
The Canceled/Executed/Vetoed states are final because they cannot be changed once they are set.
The Defeated state is also a final state because no new votes will be cast (stateInternal() may return Defeated only if the objectionPeriodEndBlock is passed).

But the Expired state depends on the GRACE_PERIOD of the timelock, and GRACE_PERIOD may be changed due to upgrades. Once the GRACE_PERIOD of the timelock is changed, the state of the proposal may also be changed, so Expired is not the final state.
```solidity
        } else if (block.timestamp >= proposal.eta + getProposalTimelock(ds, proposal).GRACE_PERIOD()) {
            return NounsDAOStorageV3.ProposalState.Expired;
        } else {
            return NounsDAOStorageV3.ProposalState.Queued;
```
Consider the following scenario.
Alice submits proposal A to stake 20,000 ETH to a DEFI protocol, and it is successfully passed, but it cannot be executed because there is now only 15,000 ETH in the timelock (consumed by other proposals), and then proposal A expires.
The DEFI protocol has been hacked or rug-pulled.
Now proposal B is about to be executed to upgrade the timelock and extend GRACE_PERIOD (e.g., GRACE_PERIOD is extended by 7 days from V1 to V2).
Alice wants to cancel Proposal A, but it cannot be canceled because it is in Expired state.
Proposal B is executed, causing Proposal A to change from Expired to Queued. 
The malicious user sends 5000 ETH to the timelock and immediately executes Proposal A to send 20000 ETH to the hacked protocol.


## Proof of Concept
https://github.com/nounsDAO/nouns-monorepo/blob/718211e063d511eeda1084710f6a682955e80dcb/packages/nouns-contracts/contracts/governance/NounsDAOV3Proposals.sol#L571-L581
## Tools Used
None
## Recommended Mitigation Steps
Consider adding a proposal expiration time field in the Proposal structure
```diff
    function queue(NounsDAOStorageV3.StorageV3 storage ds, uint256 proposalId) external {
        require(
            stateInternal(ds, proposalId) == NounsDAOStorageV3.ProposalState.Succeeded,
            'NounsDAO::queue: proposal can only be queued if it is succeeded'
        );
        NounsDAOStorageV3.Proposal storage proposal = ds._proposals[proposalId];
        INounsDAOExecutor timelock = getProposalTimelock(ds, proposal);
        uint256 eta = block.timestamp + timelock.delay();
        for (uint256 i = 0; i < proposal.targets.length; i++) {
            queueOrRevertInternal(
                timelock,
                proposal.targets[i],
                proposal.values[i],
                proposal.signatures[i],
                proposal.calldatas[i],
                eta
            );
        }
        proposal.eta = eta;
+       proposal.exp = eta + timelock.GRACE_PERIOD();
...
-       } else if (block.timestamp >= proposal.eta + getProposalTimelock(ds, proposal).GRACE_PERIOD()) {
+       } else if (block.timestamp >= proposal.exp) {
            return NounsDAOStorageV3.ProposalState.Expired;
```





## Assessed type

Context