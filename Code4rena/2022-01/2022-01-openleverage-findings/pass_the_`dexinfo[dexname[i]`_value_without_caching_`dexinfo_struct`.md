## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [pass the `dexInfo[dexName[i]` value without caching `DexInfo struct`](https://github.com/code-423n4/2022-01-openleverage-findings/issues/71) 

# Handle

rfa


# Vulnerability details

## Impact
expensive gas
## Proof of Concept
https://github.com/code-423n4/2022-01-openleverage/blob/main/openleverage-contracts/contracts/dex/bsc/BscDexAggregatorV1.sol#L47


## Recommended Mitigation Steps
replace the 2 lines of code by just 1 line:
```
dexInfo[dexName[i]] = DexInfo(factoryAddr[i], fees[i]);
```

