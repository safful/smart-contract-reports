## Tags

- bug
- disagree with severity
- 3 (High Risk)
- sponsor confirmed
- filed
- addressed

# [Proposals can be cancelled](https://github.com/code-423n4/2021-04-vader-findings/issues/227) 

# Handle

@cmichelio


# Vulnerability details


## Vulnerability Details

Anyone can cancel any proposals by calling `DAO.cancelProposal(id, id)` with `oldProposalID == newProposalID`.
This always passes the minority check as the proposal was approved.

## Impact

An attacker can launch a denial of service attack on the DAO governance and prevent any proposals from being executed.

## Recommended Mitigation Steps

Check `oldProposalID == newProposalID`


