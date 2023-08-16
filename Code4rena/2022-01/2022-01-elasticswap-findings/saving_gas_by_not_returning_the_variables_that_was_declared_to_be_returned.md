## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [saving gas by not returning the variables that was declared to be returned](https://github.com/code-423n4/2022-01-elasticswap-findings/issues/171) 

# Handle

OriDabush


# Vulnerability details

## Mathlib.sol (`calculateAddQuoteTokenLiquidityQuantities()`)
In the `calculateAddQuoteTokenLiquidityQuantities()` function, the return line is unnecessary, because those variables are returned anyway (it will save gas if you'll remove the return line.)

```sol
return (quoteTokenQty, liquidityTokenQty); // remove this line to save gas
```

