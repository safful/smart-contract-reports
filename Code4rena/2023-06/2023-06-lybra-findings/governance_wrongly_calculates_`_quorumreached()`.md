## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-08

# [Governance wrongly calculates `_quorumReached()`](https://github.com/code-423n4/2023-06-lybra-findings/issues/14) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/26915a826c90eeb829863ec3851c3c785800594b/contracts/lybra/governance/LybraGovernance.sol#L60


# Vulnerability details

## Impact
For some reason it is calculated as sum of againstVotes and abstainVotes instead of totalVotes on proposal. As the result, quorum will be reached only if >=1/3 of all votes are abstain or against, which doesn't make sense.

## Proof of Concept
Number of votes with support = 1 and support = 2 is summed up
```solidity
    function _quorumReached(uint256 proposalId) internal view override returns (bool){
        return proposalData[proposalId].supportVotes[1] + proposalData[proposalId].supportVotes[2] >= quorum(proposalSnapshot(proposalId));
    }
```
However support = 1 means against votes, support = 2 means abstain votes:
https://github.com/code-423n4/2023-06-lybra/blob/26915a826c90eeb829863ec3851c3c785800594b/contracts/lybra/governance/LybraGovernance.sol#L120-L122
```solidity
    function proposals(uint256 proposalId) external view returns (...) {
        ...
        
        forVotes =  proposalData[proposalId].supportVotes[0];
        againstVotes =  proposalData[proposalId].supportVotes[1];
        abstainVotes =  proposalData[proposalId].supportVotes[2];

        ...
    }
```

## Tools Used
Manual review

## Recommended Mitigation Steps
Use totalVotes:
```solidity
    function _quorumReached(uint256 proposalId) internal view override returns (bool){
        return proposalData[proposalId].totalVotes >= quorum(proposalSnapshot(proposalId));
    }
```


## Assessed type

Governance