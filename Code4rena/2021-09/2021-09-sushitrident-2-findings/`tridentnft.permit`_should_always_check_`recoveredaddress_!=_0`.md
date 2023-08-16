## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed

# [`TridentNFT.permit` should always check `recoveredAddress != 0`](https://github.com/code-423n4/2021-09-sushitrident-2-findings/issues/44) 

# Handle

cmichel


# Vulnerability details

The `TridentNFT.permit` function ignores the `recoveredAddress != 0` check if `isApprovedForAll[owner][recoveredAddress]` is true.

## Impact
If a user accidentally set the zero address as the operator, tokens can be stolen by anyone as a wrong signature yield `recoveredAddress == 0`.

## Recommended Mitigation Steps
Change the `require` logic to `recoveredAddress != address(0) && (recoveredAddress == owner) || isApprovedForAll[owner][recoveredAddress])`.


