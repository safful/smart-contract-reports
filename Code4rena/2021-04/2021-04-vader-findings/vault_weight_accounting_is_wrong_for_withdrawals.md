## Tags

- bug
- disagree with severity
- 3 (High Risk)
- sponsor confirmed
- filed

# [Vault Weight accounting is wrong for withdrawals](https://github.com/code-423n4/2021-04-vader-findings/issues/224) 

# Handle

@cmichelio


# Vulnerability details


## Vulnerability Details

When depositing two different synths, their weight is added to the same `mapMember_weight[_member]` storage variable.
When withdrawing the full amount of one synth with `_processWithdraw(synth, member, basisPoints=10000` the full weight is decreased.

The second deposited synth is now essentially weightless.

## Impact

Users that deposited more than one synth can not claim their fair share of rewards after a withdrawal.

## Recommended Mitigation Steps

The weight should be indexed by the synth as well.


