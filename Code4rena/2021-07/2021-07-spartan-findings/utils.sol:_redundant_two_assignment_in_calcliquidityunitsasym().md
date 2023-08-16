## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Utils.sol: Redundant two assignment in calcLiquidityUnitsAsym()](https://github.com/code-423n4/2021-07-spartan-findings/issues/63) 

# Handle

hickuphh3


# Vulnerability details

### Impact

The `calcLiquidityUnitsAsym()` function's last 2 lines are:

```jsx
uint two = 2;
return (totalSupply * amount) / (two * (amount + baseAmount));
```

The `two` assignment seems unnecessary.

### Recommended Mitigation Steps

`return (totalSupply * amount) / (two * (amount + baseAmount));`

