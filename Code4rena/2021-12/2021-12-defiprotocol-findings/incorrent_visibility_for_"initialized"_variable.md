## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Incorrent visibility for "initialized" variable](https://github.com/code-423n4/2021-12-defiprotocol-findings/issues/50) 

# Handle

neslinesli93


# Vulnerability details

## Impact
The `initialized` variable has its visibility set to `public`, while it should be private. The reason is that any contract that inherits from `Auction.sol` or `Basket.sol` may reset the value for  the `initialized` variable

## Recommended Mitigation Steps
Reduce `initialized` visibility to `private` in both contracs

