## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [Gas saving](https://github.com/code-423n4/2022-01-timeswap-findings/issues/51) 

# Handle

0x1f8b


# Vulnerability details

## Impact
Gas saving.

## Proof of Concept
Moving the substraction inside the if condition in `TimeswapPair.withdraw` could be avoided the zero substraction.

## Tools Used
Manual review.

## Recommended Mitigation Steps
Change:
```
pool.state.reserves.asset -= tokensOut.asset;
        pool.state.reserves.collateral -= tokensOut.collateral;

        if (tokensOut.asset > 0) asset.safeTransfer(assetTo, tokensOut.asset);
        if (tokensOut.collateral > 0) collateral.safeTransfer(collateralTo, tokensOut.collateral);.
```
to
```
        if (tokensOut.asset > 0) { pool.state.reserves.asset -= tokensOut.asset; asset.safeTransfer(assetTo, tokensOut.asset); }
        if (tokensOut.collateral > 0) { pool.state.reserves.collateral -= tokensOut.collateral; collateral.safeTransfer(collateralTo, tokensOut.collateral);. }
```

