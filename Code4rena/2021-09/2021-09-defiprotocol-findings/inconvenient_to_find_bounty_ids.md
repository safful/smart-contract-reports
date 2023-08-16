## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Inconvenient to find bounty ids](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/202) 

# Handle

pauliax


# Vulnerability details

## Impact
In function settleAuction user needs to decide what bounties he/she wants to claim:
    function settleAuction(
        uint256[] memory bountyIDs
    ...
    withdrawBounty(bountyIDs);
but bounties are stored in a private variable:
   Bounty[] private _bounties;
and there are no getter (view) functions to view bounties so I think that makes it very inconvenient for the end-user to find the appropriate ids that are relevant, especially considering there could be SPAM bounties as anyone can call addBounty.

## Recommended Mitigation Steps
Consider exposing public view functions to view bounties.

