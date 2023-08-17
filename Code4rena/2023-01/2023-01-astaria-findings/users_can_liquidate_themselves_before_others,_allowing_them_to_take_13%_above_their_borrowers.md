## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-20

# [Users can liquidate themselves before others, allowing them to take 13% above their borrowers](https://github.com/code-423n4/2023-01-astaria-findings/issues/281) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/AstariaRouter.sol#L611-L619


# Vulnerability details

## Impact
The `canLiquidate()` function allows liquidations to take place if either (a) the loan is over and still exists or (b) the caller owns the collateral.

In the second case, due to the liquidation fee (currently 13%), this can give a borrower an unfair position to be able to reclaim a percentage of the liquidation that should be going to their lenders.

## Proof of Concept
- A borrower puts up a piece of collateral and takes a loan of 10 WETH
- The collateral depreciates in value and they decide to keep the 10 WETH
- Right before the loans expire, the borrower can call `liquidate()` themselves
- This sets them as the `liquidator` and gives them the first 13% return on the auction
- While the lenders are left at a loss, the borrower gets to keep the 10 WETH and get a 1.3 WETH bonus

## Tools Used

Manual Review

## Recommended Mitigation Steps

Don't allow users to liquidate their own loans until they are liquidatable by the public.