## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-07

# [`_voteSucceeded()` returns true when `againstVotes > forVotes` and vice versa](https://github.com/code-423n4/2023-06-lybra-findings/issues/15) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/26915a826c90eeb829863ec3851c3c785800594b/contracts/lybra/governance/LybraGovernance.sol#L67


# Vulnerability details

## Impact
As a result voting process is broken, as it won't execute proposals with most of `forVotes`. But instead it will execute proposals with most of `againstVotes`.

## Proof of Concept
It returns whether number of votes with support = 1 is greater than with support = 0
```solidity
    function _voteSucceeded(uint256 proposalId) internal view override returns (bool){
        return proposalData[proposalId].supportVotes[1] > proposalData[proposalId].supportVotes[0];
    }
```

However support = 1 means `againstVotes`, and support = 0 means `forVotes`:
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
Manual Review

## Recommended Mitigation Steps
Swap 1 and 0:
```solidity
    function _voteSucceeded(uint256 proposalId) internal view override returns (bool){
        return proposalData[proposalId].supportVotes[0] > proposalData[proposalId].supportVotes[1];
    }
```


## Assessed type

Governance