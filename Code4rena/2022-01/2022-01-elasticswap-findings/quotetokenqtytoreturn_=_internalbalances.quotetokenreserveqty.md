## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [quoteTokenQtyToReturn = internalBalances.quoteTokenReserveQty](https://github.com/code-423n4/2022-01-elasticswap-findings/issues/176) 

# Handle

pauliax


# Vulnerability details

## Impact
Would be cheapier to have >= here when quoteTokenQtyToReturn = internalBalances.quoteTokenReserveQty to skip math operation:
```solidity
    // We should ensure no possible overflow here.
    if (quoteTokenQtyToReturn > internalBalances.quoteTokenReserveQty) {
        internalBalances.quoteTokenReserveQty = 0;
    } else {
        internalBalances.quoteTokenReserveQty -= quoteTokenQtyToReturn;
    }
```


