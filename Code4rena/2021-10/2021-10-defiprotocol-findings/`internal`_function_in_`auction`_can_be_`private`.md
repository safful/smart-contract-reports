## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [`internal` function in `Auction` can be `private`](https://github.com/code-423n4/2021-10-defiprotocol-findings/issues/15) 

# Handle

pants


# Vulnerability details

The `internal` function `Auction.withdrawBounty()` is never called by a contract that inherits `Auction`. Therefore, its visibility can be reduced to `private`.

## Impact
`private` functions are cheaper than `internal` functions.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Define this function as `private`.

