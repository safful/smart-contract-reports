## Tags

- bug
- G (Gas Optimization)
- sponsor acknowledged
- sponsor confirmed

# [Controller: Extra sload of _vaultDetails[_vault].balance](https://github.com/code-423n4/2021-09-yaxis-findings/issues/65) 

# Handle

hickuphh3


# Vulnerability details

### Impact

`_vaultDetails[_vault].balance` in L367 can be changed to the already fetched value `_balance`.

### Recommended Mitigation Steps

`_vaultDetails[_vault].balance = _vaultDetails[_vault].balance.sub(_amount);`

