## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- M-03

# [liquidateAccount will fail if transaction is not included in current block](https://github.com/code-423n4/2023-05-venus-findings/issues/365) 

# Lines of code

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/Comptroller.sol#L690-L693


# Vulnerability details

## Impact
Functon **liquidateAccount** will fail if the transaction is not included in current block because interest accures per block, and **repayAmount** and **borrowBalance** need to match precisely.

## Proof of Concept
At the end of function **liquidateAccount**, a check is performed to ensure that the **borrowBalance** is zero:

    for (uint256 i; i < marketsCount; ++i) {
            (, uint256 borrowBalance, ) = _safeGetAccountSnapshot(borrowMarkets[i], borrower);
            require(borrowBalance == 0, "Nonzero borrow balance after liquidation");
    }
This means that **repayAmount** specified in calldata must exactly match the **borrowBalance**. (If repayAmount is greater than borrowBalance, Comptroller.preLiquidateHook will revert with error TooMuchRepay.) However, the **borrowBalance** is updated every block due to interest accrual. The liquidator cannot be certain that their transaction will be included in the current block or in a future block. This uncertainty significantly increases the likelihood of liquidation failure.

## Tools Used
Manual

## Recommended Mitigation Steps
Use a looser check

    snapshot = _getCurrentLiquiditySnapshot(borrower, _getLiquidationThreshold);
    require (snapshot.shortfall == 0);
to replace

    for (uint256 i; i < marketsCount; ++i) {
            (, uint256 borrowBalance, ) = _safeGetAccountSnapshot(borrowMarkets[i], borrower);
            require(borrowBalance == 0, "Nonzero borrow balance after liquidation");
    }


## Assessed type

Invalid Validation