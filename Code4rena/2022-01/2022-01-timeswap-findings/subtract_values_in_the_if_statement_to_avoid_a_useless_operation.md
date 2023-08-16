## Tags

- bug
- G (Gas Optimization)
- resolved
- sponsor confirmed

# [subtract values in the if statement to avoid a useless operation](https://github.com/code-423n4/2022-01-timeswap-findings/issues/124) 

# Handle

OriDabush


# Vulnerability details

# TimeswapPair.sol - Gas Optimization
Lines 289-290 can be transferred into the if statements to avoid subtract 0 from the variables.

### code before:
pool.state.reserves.asset -= tokensOut.asset;

pool.state.reserves.collateral -= tokensOut.collateral;

if (tokensOut.asset > 0) asset.safeTransfer(assetTo, tokensOut.asset);

if (tokensOut.collateral > 0) collateral.safeTransfer(collateralTo, tokensOut.collateral);

### code after:

if (tokensOut.asset > 0) {
    asset.safeTransfer(assetTo, tokensOut.asset);
    pool.state.reserves.asset -= tokensOut.asset;
}

if (tokensOut.collateral > 0) {
    collateral.safeTransfer(collateralTo, tokensOut.collateral);
    pool.state.reserves.collateral -= tokensOut.collateral;
}




