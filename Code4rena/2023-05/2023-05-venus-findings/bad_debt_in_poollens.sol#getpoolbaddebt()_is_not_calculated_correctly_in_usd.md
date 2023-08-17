## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-05

# [Bad Debt in PoolLens.sol#getPoolBadDebt() is not calculated correctly in USD](https://github.com/code-423n4/2023-05-venus-findings/issues/316) 

# Lines of code

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Lens/PoolLens.sol#L248


# Vulnerability details

## Proof of Concept

In PoolLens.sol#getPoolBadDebt(), bad debt is calculated as such: 

```
            badDebt.badDebtUsd =
                VToken(address(markets[i])).badDebt() *
                priceOracle.getUnderlyingPrice(address(markets[i]));
            badDebtSummary.badDebts[i] = badDebt;
            totalBadDebtUsd = totalBadDebtUsd + badDebt.badDebtUsd;
```

In Shortfall.sol#_startAuction(), bad debt is calculated as such:

```
        uint256[] memory marketsDebt = new uint256[](marketsCount);
        auction.markets = new VToken[](marketsCount);

        for (uint256 i; i < marketsCount; ++i) {
            uint256 marketBadDebt = vTokens[i].badDebt();

            priceOracle.updatePrice(address(vTokens[i]));
            uint256 usdValue = (priceOracle.getUnderlyingPrice(address(vTokens[i])) * marketBadDebt) / 1e18;

            poolBadDebt = poolBadDebt + usdValue;
```

Focus on the line with the priceOracle.getUnderlyingPrice. In PoolLens.sol#getPoolBadDebt, badDebt in USD is calculated by multiplying the bad debt of the VToken market by the underlying price. However, in Shortfall, badDebt in USD is calculated by the bad debt of the VToken market by the underlying price and divided by 1e18. 

The PoolLens#getPoolBadDebt() function forgot to divide the debt in usd by 1e18.

This is what the function is actually counting: 

Let's say that the VToken market has a badDebt of 1.3 ETH (1e18 ETH). The pool intends to calculate 1.3 ETH in terms of USD, so it calls the oracle to determine the price of ETH. Let's say the price of ETH is 1500 USD. The total pool debt should be 1.3 * 1500 = 1950 USD. In decimal calculation, the pool debt should be 1.3e18 * 1500e18 (if oracle returns in 18 decimal places) / 1e18 = 1950e18.

## Impact

The badDebt in USD in PoolLens.sol#getPoolBadDebt() will be massively inflated.

## Tools Used

VSCode

## Recommended Mitigation Steps

Normalize the decimals of the bad debt calculation in getPoolBadDebt().

```
            badDebt.badDebtUsd =
                VToken(address(markets[i])).badDebt() *
+               priceOracle.getUnderlyingPrice(address(markets[i])) / 1e18;
            badDebtSummary.badDebts[i] = badDebt;
            totalBadDebtUsd = totalBadDebtUsd + badDebt.badDebtUsd;
```


## Assessed type

Decimal