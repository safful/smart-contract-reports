## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed
- filed

# [Canceled proposals can still be executed](https://github.com/code-423n4/2021-04-vader-findings/issues/228) 

# Handle

@cmichelio


# Vulnerability details


## Vulnerability Details

Proposals that passed the threshold ("finalized") can be cancelled by a minority again using the `cancelProposal` functions.
It only sets `mapPID_votes` to zero but `mapPID_timeStart` and `mapPID_finalising` stay the same and pass the checks in `finaliseProposal` which queues them for execution.

## Impact

Proposals cannot be cancelled.

## Recommended Mitigation Steps

Set a cancel flag and check for it in `finaliseProposal` and in execution.


