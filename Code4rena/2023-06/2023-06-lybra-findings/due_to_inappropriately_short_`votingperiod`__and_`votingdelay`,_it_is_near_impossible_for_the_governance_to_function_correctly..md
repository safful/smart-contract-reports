## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- edited-by-warden
- M-16

# [Due to inappropriately short `votingPeriod`  and `votingDelay`, it is near impossible for the governance to function correctly.](https://github.com/code-423n4/2023-06-lybra-findings/issues/268) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/governance/LybraGovernance.sol#L143-L149


# Vulnerability details

## Impact
Due to inappropriate short `votingPeriod`  and `votingDelay`, it is near impossible for the governance to function correctly.

## Proof of Concept
When making proposals with the [`Governor`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/governance/Governor.sol#L299-L308) contract OZ uses `votingPeriod`
```jsx
        uint256 snapshot = currentTimepoint + votingDelay();
        uint256 duration = votingPeriod();

        _proposals[proposalId] = ProposalCore({
            proposer: proposer,
            voteStart: SafeCast.toUint48(snapshot),//@audit votingDelay() for when the voting starts
            voteDuration: SafeCast.toUint32(duration),//@audit votingPeriod() for the duration
            executed: false,
            canceled: false
        });
``` 
But currently Lybra has implemented wrong amounts for bolt [`votingPeriod`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/governance/LybraGovernance.sol#L143-L145) and [`votingDelay`](https://github.com/code-423n4/2023-06-lybra/blob/main/contracts/lybra/governance/LybraGovernance.sol#L147-L149), which means proposals from the governance will be near impossible to be voted on.
```jsx
    function votingPeriod() public pure override returns (uint256){
         return 3;//@audit this should be time in blocks 
    }

     function votingDelay() public pure override returns (uint256){
         return 1;//@audit this should be time in blocks 
    }
```
## HH PoC
https://gist.github.com/0x3b33/dfd5a29d5fa50a00a149080280569d12

## Tools Used
Manual Review

## Recommended Mitigation Steps
You can implement it as OZ suggests in their [examples](https://docs.openzeppelin.com/contracts/4.x/governance)
```jsx
    function votingDelay() public pure override returns (uint256) {
        return 7200; // 1 day
    }

    function votingPeriod() public pure override returns (uint256) {
        return 50400; // 1 week
    }
```






## Assessed type

Governance