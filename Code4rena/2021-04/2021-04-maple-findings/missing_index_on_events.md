## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- resolved

# [Missing index on events](https://github.com/code-423n4/2021-04-maple-findings/issues/86) 

# Handle

@cmichelio


# Vulnerability details

## Vulnerability Details

Some events have no index:
- `BasicFDT.PointsCorrectionUpdated`
- `BasicFDT.PointsCorrectionUpdated`
- `BasicFDT.LossesCorrectionUpdated`
- `ExtendedFDT.LossesCorrectionUpdated`
- `StakeLocker.StakeDateUpdated`
- `MapleTreasury.FundsTokenModified` is never used


## Impact

Off-chain scripts that rely on these events are unable to filter them efficiently.

## Recommended Mitigation Steps

Add the missing indexes on the events or remove the events if they are not needed on the backend.


