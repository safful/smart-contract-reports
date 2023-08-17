## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-07

# [totalVotingPower needs to be snapshotted for each proposal because it can change and thereby affect consensus when accepting / vetoing proposals](https://github.com/code-423n4/2023-04-party-findings/issues/9) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernance.sol#L598-L605
https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/proposals/VetoProposal.sol#L46-L51


# Vulnerability details

## Impact
This issue does not manifest itself in a limited segment of the code.  

Instead it spans multiple contracts and derives its impact from the interaction of these contracts.  

In the PoC section I will do my best in explaining how this results in an issue.  

I discussed this with the sponsor and they explained to me that this issue is due to a PR that has unintentionally not been merged.  

![Discord message](https://user-images.githubusercontent.com/118979828/231990051-b9f731f1-1678-43e3-81e4-7ec0164bdc10.png)


So they have already written the code that is necessary to fix this issue. It's just not been merged with this branch. So since the sponsor knows about this already and it's just the PR that has gone missing it's not necessary for me to provide the full Solidity code to fix this issue.  

In short, this issue is due to the fact that the `totalVotingPower` is not snapshotted when a proposal is created.  

The votes that are used to vote for a proposal (or veto it) are based on a specific snapshot (1 block prior to the proposal being created).  

When the `totalVotingPower` changes this leads to unintended consequences.  

When `totalVotingPower` decreases, votes become more valuable than they should be.  

And when `totalVotingPower` increases, votes become less valuable than they should be.  

## Proof of Concept
When a proposal is created via the [`PartyGovernance.propose`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernance.sol#L527-L548) function, the proposal's `proposedTime` is set:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernance.sol#L537-L543)  
```solidity
ProposalStateValues({
    proposedTime: uint40(block.timestamp),
    passedTime: 0,
    executedTime: 0,
    completedTime: 0,
    votes: 0
}),
```

When users then vote in order to accept the proposal or veto the proposal, their votes are based on the snapshot at the `proposedTime - 1` timestamp.  

We can see this in the `PartyGovernance.accept` function:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernance.sol#L592)  
```solidity
uint96 votingPower = getVotingPowerAt(msg.sender, values.proposedTime - 1, snapIndex);
```

And we can see it in the `VetoProposal.voteToVeto` function:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/proposals/VetoProposal.sol#L37-L41)  
```solidity
uint96 votingPower = party.getVotingPowerAt(
    msg.sender,
    proposalValues.proposedTime - 1,
    snapIndex
);
```

However the `totalVotingPower` to determine whether enough votes have been collected is the current `totalVotingPower`:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/party/PartyGovernance.sol#L598-L605)  
```solidity
if (
    values.passedTime == 0 &&
    _areVotesPassing(
        values.votes,
        _governanceValues.totalVotingPower,
        _governanceValues.passThresholdBps
    )
) {
```

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/proposals/VetoProposal.sol#L46-L51)  
```solidity
if (
    _areVotesPassing(
        newVotes,
        governanceValues.totalVotingPower,
        governanceValues.passThresholdBps
    )
```

The `totalVotingPower` is not constant. It can increase and decrease.  

Now we can understand the issue. The `totalVotingPower` must be based on the same time as the votes (i.e. `proposedTime - 1`).  

Let's look at a scenario:  

```
At the time of proposal creation (proposedTime - 1):

Alice: 100 Votes
Bob: 50 Votes
Chris: 50 Votes

totalVotingPower=200
```

Let's say 80% of votes are necessary for the proposal to pass.  

Now the `totalVotingPower` is increased (e.g. by a `ReraiseETHCrowdfund`) since David now has 100 Votes:  

```
Alice: 100 Votes
Bob: 50 Votes
Chris: 50 Votes
David: 100 Votes 

totalVotingPower=300
```

Now it is impossible for the proposal to pass.  

The proposal needs 80% of 300 Votes which is 240 Votes. But the votes are used from the old snapshot and there were only 200 Votes.  

The old `totalVotingPower` should have been used (200 Votes instead of 300 Votes).  

Similarly there is an issue when `totalVotingPower` decreases:  

```
Alice: 100 Votes
Bob: 50 Votes
Chris: 0 Votes

totalVotingPower=150
```

If 60% of the votes are necessary for the proposal to pass, Alice can make the proposal pass on her own because `totalVotingPower=150` is used even though the old `totalVotingPower=200` should be used.  


## Tools Used
VSCode

## Recommended Mitigation Steps
As explained above the sponsor already has the code to implement snapshotting the `totalVotingPower`.  

In short the following changes need to be made:  

1. snapshot `totalVotingPower` whenever it is changed

2. Whenever `totalVotingPower` is used to calculate whether a proposal is accepted / vetoed, the snapshot should be used

