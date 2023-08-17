## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- edited-by-warden
- M-12

# [VetoProposal: proposals cannot be vetoed in all states in which it should be possible to veto proposals](https://github.com/code-423n4/2023-04-party-findings/issues/3) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/proposals/VetoProposal.sol#L19-L60


# Vulnerability details

## Impact
The [`VetoProposal`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/proposals/VetoProposal.sol#L8-L69) contract allows to veto proposals with the [`voteToVeto`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/proposals/VetoProposal.sol#L19-L60) function.  

The proposal can only be vetoed when it is in the `Voting` state, otherwise the `voteToVeto` function reverts.  

The issue is that the `Voting` state is not the only state in which it should be possible to veto the proposal. It should also be possible to veto the proposal in the `Passed` and `Ready` states.  

(We can see this by looking at the downstream [`PartyGovernance.veto`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernance.sol#L617-L637) function)

It has been confirmed to me by the sponsor that the `voteToVeto` function should not restrict the situations in which vetos can occur.  

The impact of this issue is that the situations in which vetos can occur is more limited than it should be. Users should have the ability to veto proposals even in the `Passed` and `Ready` states but they don't.  

## Proof of Concept
By looking at the `VetoProposal.voteToVeto` function we see that it's only possible to call the function when the proposal is in the `Voting` state. Otherwise the function reverts:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/proposals/VetoProposal.sol#L28-L34)  
```solidity
// Check that proposal is active
(
    PartyGovernance.ProposalStatus proposalStatus,
    PartyGovernance.ProposalStateValues memory proposalValues
) = party.getProposalStateInfo(proposalId);
if (proposalStatus != PartyGovernance.ProposalStatus.Voting)
    revert ProposalNotActiveError(proposalId);
```

But when we look at the `PartyGovernance.veto` function which is called downstream and which implements the actual veto functionality (the `VetoProposal.voteToVeto` function is only a wrapper) we can see that it allows vetoing in the `Voting`, `Passed` and `Ready` states:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernance.sol#L617-L637)  
```solidity
function veto(uint256 proposalId) external onlyHost onlyDelegateCall {
    // Setting `votes` to -1 indicates a veto.
    ProposalState storage info = _proposalStateByProposalId[proposalId];
    ProposalStateValues memory values = info.values;


    {
        ProposalStatus status = _getProposalStatus(values);
        // Proposal must be in one of the following states.
        if (
            status != ProposalStatus.Voting &&
            status != ProposalStatus.Passed &&
            status != ProposalStatus.Ready
        ) {
            revert BadProposalStatusError(status);
        }
    }


    // -1 indicates veto.
    info.values.votes = VETO_VALUE;
    emit ProposalVetoed(proposalId, msg.sender);
}
```

Therefore we can see that the `VetoProposal.voteToVeto` function restricts the vetoing functionality too much.  

Users are not able to veto in the `Passed` and `Ready` states even though it should be possible.  

## Tools Used
VSCode

## Recommended Mitigation Steps
The issue can be fixed by allowing the `VetoProposal.voteToVeto` function to be called in the `Passed` and `Ready` states as well.  

Fix:  
```diff
diff --git a/contracts/proposals/VetoProposal.sol b/contracts/proposals/VetoProposal.sol
index 780826f..38410f6 100644
--- a/contracts/proposals/VetoProposal.sol
+++ b/contracts/proposals/VetoProposal.sol
@@ -30,7 +30,11 @@ contract VetoProposal {
             PartyGovernance.ProposalStatus proposalStatus,
             PartyGovernance.ProposalStateValues memory proposalValues
         ) = party.getProposalStateInfo(proposalId);
-        if (proposalStatus != PartyGovernance.ProposalStatus.Voting)
+        if (
+            proposalStatus != PartyGovernance.ProposalStatus.Voting
+            && proposalStatus != PartyGovernance.ProposalStatus.Passed
+            && proposalStatus != PartyGovernance.ProposalStatus.Ready
+           )
             revert ProposalNotActiveError(proposalId);
```



