## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed

# [`LPToken.set_minter()` doesn't check that `_minter` doesn't equal zero](https://github.com/code-423n4/2021-11-bootfinance-findings/issues/69) 

# Handle

pants


# Vulnerability details

The function `LPToken.set_minter()` doesn't check that `_minter` doesn't equal zero before it sets it as the new minter.

## Impact
This function can be invoked by mistake with the zero address as `_minter`, causing the system to lose its minter forever, without the option to set a new minter.

## Tool Used
Manual code review.

## Recommended Mitigation Steps
Check that `_minter` doesn't equal zero before setting it as the new minter.

