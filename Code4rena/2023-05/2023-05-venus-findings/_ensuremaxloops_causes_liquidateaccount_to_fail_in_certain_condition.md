## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-04

# [_ensureMaxLoops causes liquidateAccount to fail in certain condition](https://github.com/code-423n4/2023-05-venus-findings/issues/327) 

# Lines of code

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L667


# Vulnerability details

## Impact
The function **_ensureMaxLoops** reverts if the iteration count exceeds the **maxLoopsLimit**. However, the limitation imposed by **maxLoopsLimit** hinders the functioning of **liquidateAccount** under certain conditions, as **orderCount** needs to reach twice the market count (which is also constrained by the **maxLoopsLimit**) in extreme cases.

## Proof of Concept
Suppose **maxLoopsLimit** is set to **16** and currently **12** markets has been added, which is allowed by **_ensureMaxLoops** in function **_addMarket**:

    allMarkets.push(VToken(vToken));
    marketsCount = allMarkets.length;
    _ensureMaxLoops(marketsCount);
Then, Alice enters all the **12** markets by depositing and borrowing simultaneously, which is also allowed by **_ensureMaxLoops** in function **enterMarkets**:

    uint256 len = vTokens.length;
    uint256 accountAssetsLen = accountAssets[msg.sender].length;
    _ensureMaxLoops(accountAssetsLen + len);
To illustrate, assume these **12** coins are all stablecoin with an equal value. Let's call them USDA, USDB, USDC,..., USDL. Alice deposits 20 USDA, 1.1USDB, 1.1USDC,..., 1.1USDL, worth 32.1USD in total, then she borrows 2USDA, 2USDB, 2USDC,..., 2USDL, worth 24 USD in total. Unluckily, USDA depegs to 0.6USD, Alice's deposit value drop to 24.1USD, which is below the liquidation threshold (also below the minLiquidatableCollateral). However, nobody can liquidate Alice's account by calling **liquidateAccount**, because the least possible **orderCount** is **23**, which exceeds **maxLoopsLimit**.

Let's take a closer look at **LiquidationOrder**:

    struct LiquidationOrder {
           VToken vTokenCollateral;
           VToken vTokenBorrowed;
           uint256 repayAmount;
    }
In this case, liquidator cannot perfectly match **vTokenCollateral** with **vTokenBorrowed** one-to-one. Because the value of collateral and debt is not equal, more than one order is needed to liquidate each asset. To generalize, if asset count is **n**, in the worst case, **2n-1** orders are needed for a complete liquidation (not hard to prove).

## Tools Used
Manual

## Recommended Mitigation Steps
    _ensureMaxLoops(ordersCount / 2);



## Assessed type

Loop