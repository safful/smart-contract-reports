## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [`TridentNFT.permitAll` prviliges discrepancy for operator](https://github.com/code-423n4/2021-09-sushitrident-2-findings/issues/45) 

# Handle

cmichel


# Vulnerability details

The `TridentNFT.permitAll` function allows the operator (`isApprovedForAll[owner][recoveredAddress]`) to change the operator (and lock themself out).
The same functionality without permits does not work as `setApprovalForAll` requires the `owner` authority.

## Impact
`permitAll` should have the same auth checks as `setApprovalForAll` and not allow the `operator` to change the operator.

## Recommended Mitigation Steps
Remove the `|| isApprovedForAll[owner][recoveredAddress]` from the `require` statement.


