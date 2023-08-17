## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- upgraded by judge
- H-07

# [Malicious refinancing attack could lead to suboptimal NFT liquidation](https://github.com/code-423n4/2023-01-astaria-findings/issues/430) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/AstariaRouter.sol#L684


# Vulnerability details

## Impact
A malicious refinancing with a very low `liquidationInitialAsk` just prior to a liquidation event could result in an NFT being liquidated for a price much lower than what the borrower wanted when signing up for the loan. Because refinancing is permission less, anyone can do this close to liquidation event resulting in the user being compensated less than fair price for their NFT. 

## Proof of Concept

Refinance checks are currently permission less, anyone can buyout a lien. This is fine because of the assumption that refinancing leads to a strictly optimal outcome in all cases to the borrower. This is true with respect to the loan duration, interest rate and overall debt parameters. However this is not the case with respect to the `liquidationInitialAsk` loan parameter.

See code in https://github.com/code-423n4/2023-01-astaria/blob/main/src/AstariaRouter.sol#L684 refinance checks do not take into account `liquidationInitialAsk` which is one of the loan parameters

Imagine a user takes a loan for 3 ETH against an NFT with a high `liquidationInitialAsk` of 100 ETH which is a fair value of the NFT to the user. If they are not able to pay off the loan in time, they are assured ~97 ETH back assuming market conditions do not change. However a malicious refinancing done close to liquidation can set `liquidationInitialAsk` close to 0.

This is possible because:
-  Refinancing is permission less
-  Since it's close to liquidation, user has no time to react

The malicious liquidator just needs to pay off the debt of 3 ETH and a minimal liquidation fee. Further, since they are aware of the initial ask in the NFT auction, they may be able to purchase the NFT for a very low price. The liquidator profits and the initial borrower does not receive a fair market value for their collateral.


## Tools Used
Just read the docs & code

## Recommended Mitigation Steps
- Add checks that `liquidationInitialAsk` does not decrease during a refinance. Or set a `minimumLiquidationPrice` which is respected across all refinances
- Don't allow refinances close to a liquidation event
- Don't allow loans / refinances less than a minimum duration. May prevent other classes of surprises as well. 