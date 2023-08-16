## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Dao.sol: Restrict Function Visibilities](https://github.com/code-423n4/2021-07-spartan-findings/issues/48) 

# Handle

hickuphh3


# Vulnerability details

### Impact

- `calcClaimBondedLP()` returns `_BONDVAULT.calcBondedLP(()` which is a view function. Hence, `calcClaimBondedLP()` can be a view function as well.
- `hasMinority()` is not called within the contract. Hence, the `public` keyword can be reduced to `external` to save gas.

### Recommended Mitigation Steps

- Restrict `calcClaimBondedLP()` visibility to `view` (ie. add `view` keyword).
- Reduce `hasMinority()` from `public` to `external`

