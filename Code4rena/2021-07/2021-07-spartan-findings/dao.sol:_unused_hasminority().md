## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [Dao.sol: Unused hasMinority()](https://github.com/code-423n4/2021-07-spartan-findings/issues/69) 

# Handle

hickuphh3


# Vulnerability details

### Impact

`hasMinority()` is defined as a public function, but is unused in the contract. It can either be entirely removed or have its visibility changed to `external`.

