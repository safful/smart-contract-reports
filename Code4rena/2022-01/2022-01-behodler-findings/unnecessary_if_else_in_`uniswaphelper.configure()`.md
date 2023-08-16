## Tags

- bug
- disagree with severity
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Unnecessary if else in `UniswapHelper.configure()`](https://github.com/code-423n4/2022-01-behodler-findings/issues/89) 

# Handle

Ruhum


# Vulnerability details

## Impact
The if else here doesn't really do anything. Might as well just remove it:
https://github.com/code-423n4/2022-01-behodler/blob/main/contracts/UniswapHelper.sol#L132

## Recommended Mitigation Steps
`VARS.precision = precision`

