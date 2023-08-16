## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [There may be no bounties or user is not interested in any of them](https://github.com/code-423n4/2021-10-defiprotocol-findings/issues/87) 

# Handle

pauliax


# Vulnerability details

## Impact
function settleAuction could skip withdrawBounty if there are no bounties.

## Recommended Mitigation Steps
if (bountyIDs.length > 0) {
  withdrawBounty(bountyIDs);
}


