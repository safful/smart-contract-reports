## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Cache and read storage variables from the stack can save gas](https://github.com/code-423n4/2022-01-elasticswap-findings/issues/156) 

# Handle

WatchPug


# Vulnerability details

For the storage variables that will be accessed multiple times, cache and read from the stack can save ~100 gas from each extra read (`SLOAD` after Berlin).

For example:

https://github.com/code-423n4/2022-01-elasticswap/blob/d107a198c0d10fbe254d69ffe5be3e40894ff078/elasticswap/src/contracts/Exchange.sol#L217-L233

```solidity
  uint256 baseTokenQtyToRemoveFromInternalAccounting =
      (_liquidityTokenQty * internalBalances.baseTokenReserveQty) /
          totalSupplyOfLiquidityTokens;

  internalBalances
      .baseTokenReserveQty -= baseTokenQtyToRemoveFromInternalAccounting;

  // We should ensure no possible overflow here.
  if (quoteTokenQtyToReturn > internalBalances.quoteTokenReserveQty) {
      internalBalances.quoteTokenReserveQty = 0;
  } else {
      internalBalances.quoteTokenReserveQty -= quoteTokenQtyToReturn;
  }

  internalBalances.kLast =
      internalBalances.baseTokenReserveQty *
      internalBalances.quoteTokenReserveQty;
```

`internalBalances.baseTokenReserveQty` and `internalBalances.quoteTokenReserveQty` can be cached.

### Recommendation

Change to:

```solidity
uint256 internalBaseTokenReserveQty = internalBalances.baseTokenReserveQty;
uint256 baseTokenQtyToRemoveFromInternalAccounting =
    (_liquidityTokenQty * internalBaseTokenReserveQty) /
        totalSupplyOfLiquidityTokens;

internalBalances
    .baseTokenReserveQty = internalBaseTokenReserveQty = internalBaseTokenReserveQty - baseTokenQtyToRemoveFromInternalAccounting;

// We should ensure no possible overflow here.
uint256 internalQuoteTokenReserveQty = internalBalances.quoteTokenReserveQty;
if (quoteTokenQtyToReturn > internalQuoteTokenReserveQty) {
    internalBalances.quoteTokenReserveQty = internalQuoteTokenReserveQty = 0;
} else {
    internalBalances.quoteTokenReserveQty = internalQuoteTokenReserveQty = internalQuoteTokenReserveQty - quoteTokenQtyToReturn;
}

internalBalances.kLast =
    internalBaseTokenReserveQty *
    internalQuoteTokenReserveQty;
```

