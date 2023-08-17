## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor confirmed
- edited-by-warden
- H-08

# [VetoProposal: user can veto multiple times so every proposal can be votoed by any user that has a small amount of votes](https://github.com/code-423n4/2023-04-party-findings/issues/2) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/proposals/VetoProposal.sol#L19-L60


# Vulnerability details

## Impact
The [`VetoProposal`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/proposals/VetoProposal.sol#L8-L69) contract allows to veto proposals with the [`voteToVeto`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/proposals/VetoProposal.sol#L19-L60) function.  

When the amount of votes collected to veto a proposal exceeds a certain threshold (the `passThresholdBps`, which is determined upon initialization of the party), the proposal is vetoed, meaning it cannot execute anymore (its status becomes `Defeated`).  

The `passThresholdBps` specifies a percentage of the `totalVotingPower` of the party.  

E.g. `passThresholdBps=1000` means that 10% of the `totalVotingPower` must veto a proposal such that the veto goes through.  

The issue is that the contract lacks the obvious check that a user has not vetoed before, thereby a user can veto multiple times.  

So say a user holds 1% of `totalVotingPower` and in order for the veto to go through, 10% of `totalVotingPower` must veto.  

The user can just veto 10 times to reach the 10% requirement.  

The impact is obvious: Any user with a small amount of votes can veto any proposal. This is a critical bug since the party may become unable to perform any actions if there is a user that vetoes all proposals.  

## Proof of Concept
Add the following test to the `VetoProposal.t.sol` test file:  

```solidity
function test_VetoMoreThanOnce() public {
    _assertProposalStatus(PartyGovernance.ProposalStatus.Voting);

    // Vote to veto
    vm.prank(voter1);
    vetoProposal.voteToVeto(party, proposalId, 0);

    _assertProposalStatus(PartyGovernance.ProposalStatus.Voting);
    assertEq(vetoProposal.vetoVotes(party, proposalId), 1e18);

    // Vote to veto (passes threshold)
    vm.prank(voter1);
    vetoProposal.voteToVeto(party, proposalId, 0);

    _assertProposalStatus(PartyGovernance.ProposalStatus.Defeated);
    assertEq(vetoProposal.vetoVotes(party, proposalId), 0); // Cleared after proposal is vetoed
}
```
In the test file, these are the conditions: `totalVotingPower = 3e18`, required votes threshold is 51%, `voter1` has `1e18` votes which is `~33%`. Clearly `voter1` should not be able to veto the proposal on his own.    

You can see in the test that `voter1` can veto 2 times.  
After the first call to `voteToVeto`, the threshold is not yet reached (the proposal is still in the `Voting` state).  

After the second call to `voteToVeto` the threshold is reached and the proposal is in the `Defeated` state.  

## Tools Used
VSCode, Foundry

## Recommended Mitigation Steps
The fix is straightforward.  

We introduce a `hasVoted` mapping that tracks for each `(party, proposalId, address)` triplet if it has vetoed already.  

Fix:  
```diff
diff --git a/contracts/proposals/VetoProposal.sol b/contracts/proposals/VetoProposal.sol
index 780826f..fb1f1ab 100644
--- a/contracts/proposals/VetoProposal.sol
+++ b/contracts/proposals/VetoProposal.sol
@@ -8,9 +8,11 @@ import "../party/Party.sol";
 contract VetoProposal {
     error NotPartyHostError();
     error ProposalNotActiveError(uint256 proposalId);
+    error AlreadyVotedError(address caller);
 
     /// @notice Mapping from party to proposal ID to votes to veto the proposal.
     mapping(Party => mapping(uint256 => uint96)) public vetoVotes;
+    mapping(Party => mapping(uint256 => mapping(address => bool))) public hasVoted;
 
     /// @notice Vote to veto a proposal.
     /// @param party The party to vote on.
@@ -33,6 +35,12 @@ contract VetoProposal {
         if (proposalStatus != PartyGovernance.ProposalStatus.Voting)
             revert ProposalNotActiveError(proposalId);
 
+        if (hasVoted[party][proposalId][msg.sender]) {
+            revert AlreadyVotedError(msg.sender);
+        }
+
+        hasVoted[party][proposalId][msg.sender] = true;
+
         // Increase the veto vote count
         uint96 votingPower = party.getVotingPowerAt(
             msg.sender,
```
