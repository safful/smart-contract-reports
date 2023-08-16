## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Dead code](https://github.com/code-423n4/2021-09-sushimiso-findings/issues/123) 

# Handle

pauliax


# Vulnerability details

## Impact
WETH state variable in MISOLauncher is practically useless as it is not used in any meaningful way. Similarly, SECONDS_PER_DAY is not used in PostAuctionLauncher.

## Recommended Mitigation Steps
Consider removing unused variables.

