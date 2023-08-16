## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed
- resolved

# [Typo NULL_TRASNFER_DST](https://github.com/code-423n4/2021-04-maple-findings/issues/25) 

# Handle

gpersoon


# Vulnerability details

## Impact

A require statement in the function transfer in LiquidityLocker.sol contains a typo.
TRASNFER should be TRANSFER

## Proof of Concept

LiquidityLocker.sol: function transfer(address dst, uint256 amt) external isPool {
LiquidityLocker.sol:    require(dst != address(0), "LiquidityLocker:NULL_TRASNFER_DST");

## Tools Used

Editor

## Recommended Mitigation Steps

Fix typo


