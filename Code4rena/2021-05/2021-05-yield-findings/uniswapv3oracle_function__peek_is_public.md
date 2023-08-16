## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [UniswapV3Oracle function _peek is public](https://github.com/code-423n4/2021-05-yield-findings/issues/37) 

# Handle

pauliax


# Vulnerability details

## Impact
In contract UniswapV3Oracle function _peek has visibility of public while the name and similar functions in other oracles are declared as private.

## Recommended Mitigation Steps
give _peek private visibility.

