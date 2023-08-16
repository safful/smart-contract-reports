## Tags

- bug
- 1 (Low Risk)
- sponsor acknowledged
- sponsor confirmed

# [Decimals of upgradeable tokens may change](https://github.com/code-423n4/2021-09-yaxis-findings/issues/82) 

# Handle

pauliax


# Vulnerability details

## Impact
A theoretical issue is that the decimals of USDC may change as they use an upgradeable contract so you cannot assume that it stays 6 decimals forever:
  balances[1] = stableSwap3Pool.balances(1).mul(10**12); // USDC

## Recommended Mitigation Steps
A simple solution would be to call .decimals() on token contract to query it on the go. Then you will not need to hardcode it but gas usage will increase.

