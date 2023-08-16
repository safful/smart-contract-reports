## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [TRIBERageQuit: Redundant oracleAddress variable](https://github.com/code-423n4/2021-11-fei-findings/issues/108) 

# Handle

hickuphh3


# Vulnerability details

## Impact

The following line

```jsx
address public constant oracleAddress =
	0xd1866289B4Bd22D453fFF676760961e0898EE9BF; // oracle with caching
```

is only used in the instantiation of the oracle

`IOracle public constant oracle = IOracle(oracleAddress);`

The first instantiation can be combined with the second to save gas.

## Recommended Mitigation Steps

`IOracle public constant oracle = IOracle(0xd1866289B4Bd22D453fFF676760961e0898EE9BF);`

