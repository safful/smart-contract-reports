## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [Inclusive conditions](https://github.com/code-423n4/2022-01-elasticswap-findings/issues/175) 

# Handle

pauliax


# Vulnerability details

## Impact
Conditions should be inclusive >= or <= :
```solidity
  require(
      baseTokenQty > _baseTokenQtyMin,
      "MathLib: INSUFFICIENT_BASE_TOKEN_QTY"
  );
  require(
      quoteTokenQty > _quoteTokenQtyMin,
      "MathLib: INSUFFICIENT_QUOTE_TOKEN_QTY"
  );
  require(
      _baseTokenQtyMin < maxBaseTokenQty,
      "MathLib: INSUFFICIENT_DECAY"
  );
  require(
      _quoteTokenQtyMin < maxQuoteTokenQty,
      "MathLib: INSUFFICIENT_DECAY"
  );
```

Otherwise, these functions will fail when e.g. baseTokenQty = _baseTokenQtyMin when the end-user expects it to pass through.

