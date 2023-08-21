## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-01

# [Wrong `proposalThreshold` amount in `LybraGovernance.sol`](https://github.com/code-423n4/2023-06-lybra-findings/issues/942) 

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/governance/LybraGovernance.sol#L172-L174


# Vulnerability details

## Impact

The proposal can be created with only 100_000 esLBR delegated instead of 10_000_000.

## Proof of Concept

According to [LybraV2Docs](https://docs.lybra.finance/lybra-v2-technical-beta/governance/process/propose#threshold), a proposal can only be created if the sender has at least 10 million esLBR tokens delegated to his address to meet the proposal threshold.

In [LybraGovernance.sol#L172-L174](https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/governance/LybraGovernance.sol#L172-L174) the proposal threshold is set to only `1e23` which equals to `100_000` as esLBR has 18 decimals.
```
    function proposalThreshold() public pure override returns (uint256) {
        return 1e23;
    }
```
[LybraGovernance.sol#L172-L174](https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/governance/LybraGovernance.sol#L172-L174)


## Tools Used
Manual Review

## Recommended Mitigation Steps
In [LybraGovernance.sol#L173](https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/governance/LybraGovernance.sol#L173) replace `1e23` with `1e25`

Alternatively, the team can update the documentation stating that it is only required 100_000 esLBR tokens (0.1% of the total LBR supply) delegated to meet the proposal threshold.



## Assessed type

Math