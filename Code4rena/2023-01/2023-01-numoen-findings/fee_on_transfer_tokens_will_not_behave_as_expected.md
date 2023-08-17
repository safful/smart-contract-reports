## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-01

# [Fee on transfer tokens will not behave as expected](https://github.com/code-423n4/2023-01-numoen-findings/issues/263) 

# Lines of code

https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/Lendgine.sol#L99


# Vulnerability details

## Impact
In Numoen, it does not specifically restrict the type of ERC20 collateral used for borrowing.

If fee on transfer token(s) is/are entailed, it will specifically make mint() revert in Lendgine.sol when checking if balanceAfter < balanceBefore + collateral.

## Proof of Concept
[File: Lendgine.sol#L71-L102](https://github.com/code-423n4/2023-01-numoen/blob/main/src/core/Lendgine.sol#L71-L102)

```solidity
  function mint(
    address to,
    uint256 collateral,
    bytes calldata data
  )
    external
    override
    nonReentrant
    returns (uint256 shares)
  {
    _accrueInterest();

    uint256 liquidity = convertCollateralToLiquidity(collateral);
    shares = convertLiquidityToShare(liquidity);

    if (collateral == 0 || liquidity == 0 || shares == 0) revert InputError();
    if (liquidity > totalLiquidity) revert CompleteUtilizationError();
    // next check is for the case when liquidity is borrowed but then was completely accrued
    if (totalSupply > 0 && totalLiquidityBorrowed == 0) revert CompleteUtilizationError();

    totalLiquidityBorrowed += liquidity;
    (uint256 amount0, uint256 amount1) = burn(to, liquidity);
    _mint(to, shares);

    uint256 balanceBefore = Balance.balance(token1);
    IMintCallback(msg.sender).mintCallback(collateral, amount0, amount1, liquidity, data);
    uint256 balanceAfter = Balance.balance(token1);

99:    if (balanceAfter < balanceBefore + collateral) revert InsufficientInputError();

    emit Mint(msg.sender, collateral, shares, liquidity, to);
  }
```
As can be seen from the code block above, line 99 is meant to be reverting when `balanceAfter < balanceBefore + collateral`. So in the case of deflationary tokens, the error is going to be thrown even though the token amount has been received due to the fee factor.

## Tools Used
Manual inspection

## Recommended Mitigation Steps
Consider:

1. whitelisting token0 and token1 ensuring no fee-on-transfer token is allowed when a new instance of a market is created using the factory, or
2. calculating the balance before and after the transfer of token1 (collateral), and use the difference between those two balances as the amount received rather than using the input amount `collateral` if deflationary token is going to be allowed in the protocol.