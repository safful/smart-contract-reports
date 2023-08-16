## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [&& operator can use more gas](https://github.com/code-423n4/2022-01-xdefi-findings/issues/128) 

# Handle

rfa


# Vulnerability details

## Impact
more expensive gas usage

## Proof of Concept
instead of using operator && on single require check (XDEFIDistribution.sol line 255). using double require check can save more gas:

 require(amount_ != uint256(0) && amount_ <= MAX_TOTAL_XDEFI_SUPPLY, "INVALID_AMOUNT");

## Tools Used

## Recommended Mitigation Steps
require(amount_ != uint256(0), "INVALID_AMOUNT" );
require(amount_ <= MAX_TOTAL_XDEFI_SUPPLY, "INVALID_AMOUNT");

