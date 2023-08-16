## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: Reorder require statements `MathLib.sol:calculateAddLiquidityQuantities()` to save gas on revert](https://github.com/code-423n4/2022-01-elasticswap-findings/issues/34) 

# Handle

Dravee


# Vulnerability details

## Impact 
Some of the require statements can be placed earlier to reduce gas usage on revert.

## Proof of Concept 
The following can be reordered to save gas on revert: 
```
File: MathLib.sol
464:                 tokenQtys.baseTokenQty += baseTokenQtyFromDecay;
465:                 tokenQtys.quoteTokenQty += quoteTokenQtyFromDecay;
466:                 tokenQtys.liquidityTokenQty += liquidityTokenQtyFromDecay;
467: 
468:                 require(
469:                     tokenQtys.baseTokenQty >= _baseTokenQtyMin,
470:                     "MathLib: INSUFFICIENT_BASE_QTY"
471:                 );
472: 
473:                 require(
474:                     tokenQtys.quoteTokenQty >= _quoteTokenQtyMin,
475:                     "MathLib: INSUFFICIENT_QUOTE_QTY"
476:                 );
```
to
```
File: MathLib.sol
464:                 tokenQtys.baseTokenQty += baseTokenQtyFromDecay;
465:                 
466:                 require(
467:                     tokenQtys.baseTokenQty >= _baseTokenQtyMin,
468:                     "MathLib: INSUFFICIENT_BASE_QTY"
469:                 );
470: 
471:                 tokenQtys.quoteTokenQty += quoteTokenQtyFromDecay;
472: 
473:                 require(
474:                     tokenQtys.quoteTokenQty >= _quoteTokenQtyMin,
475:                     "MathLib: INSUFFICIENT_QUOTE_QTY"
476:                 );
477: 
478:                 tokenQtys.liquidityTokenQty += liquidityTokenQtyFromDecay;
```

## Tools Used 
VS Code 

## Recommended Mitigation Steps 
Relocate the said require statements

