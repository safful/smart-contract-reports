## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [`removeLiquidity.sol#baseTokenQtyToRemoveFromInternalAccounting` should not be cached](https://github.com/code-423n4/2022-01-elasticswap-findings/issues/94) 

# Handle

0x0x0x


# Vulnerability details

`removeLiquidity.sol#baseTokenQtyToRemoveFromInternalAccounting` is used only once and caching it does cost extra gas.
So
```
        uint256 baseTokenQtyToRemoveFromInternalAccounting =
            (_liquidityTokenQty * internalBalances.baseTokenReserveQty) /
                totalSupplyOfLiquidityTokens;

        internalBalances
            .baseTokenReserveQty -= baseTokenQtyToRemoveFromInternalAccounting;
```
can be replaced with
```

        internalBalances.baseTokenReserveQty -= (_liquidityTokenQty * internalBalances.baseTokenReserveQty) / totalSupplyOfLiquidityTokens;
```

