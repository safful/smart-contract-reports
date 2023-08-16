## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [`Auction.settleAuction()` and `IAuction.settleAuction()` accept arguments with different data locations](https://github.com/code-423n4/2021-10-defiprotocol-findings/issues/41) 

# Handle

pants


# Vulnerability details

The function `Auction.settleAuction()` claims to override `IAuction.settleAuction()`, but their arguments have different data locations.

## Impact
Mismatching data locations in overrides have unexpected behavior.

## Proof of Concept
https://github.com/ethereum/solidity/issues/10900

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Modify the data locations of the arguments to match between `Auction.settleAuction()` and `IAuction.settleAuction()`.

