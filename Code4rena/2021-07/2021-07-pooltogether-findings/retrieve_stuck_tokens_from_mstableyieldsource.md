## Tags

- bug
- 1 (Low Risk)
- mStableYieldSource
- sponsor confirmed

# [Retrieve stuck tokens from MStableYieldSource](https://github.com/code-423n4/2021-07-pooltogether-findings/issues/73) 

# Handle

pauliax


# Vulnerability details

## Impact
Tokens sent directly to the MStableYieldSource will be stuck forever. Consider adding a function that allows an admin to retrieve stuck tokens:
* Balance of mAsset - total deposited amount of mAsset;
* Similar with credit balances as credits are issued as a separate erc20 token.
* All the other tokens.

