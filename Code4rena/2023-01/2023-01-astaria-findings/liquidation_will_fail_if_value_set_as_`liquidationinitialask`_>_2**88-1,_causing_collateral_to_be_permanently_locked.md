## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- H-10

# [Liquidation will fail if value set as `liquidationInitialAsk` > 2**88-1, causing collateral to be permanently locked](https://github.com/code-423n4/2023-01-astaria-findings/issues/375) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L340


# Vulnerability details

##### Description:

When a lien is initially created, the `liquidationInitialAsk` can be set as any uint256 value >= the amount of underlying borrowed. Later on however, if the position is liquidated, an auction is created which casts the `liquidationInitialAsk` value to a uint88. Taking a look at the function in `SafeCastLib.sol`, we can see that if the value is greater than the max uint88 value, execution is reverted:

```
function safeCastTo88(uint256 x) internal pure returns (uint88 y) {
  require(x < 1 << 88);

  y = uint88(x);
}
```

This reversion prevents auctions from ever being initialized, and since the only way to retrieve the collateral after the loan has expired is through auction, the collateral is permanently locked in the contract.

For reference, setting the `initialLiquidationAsk` > 309,485,009.8213451 DAI would trigger this error, and with a lesser value or higher decimal collateral, this may require a much lower USD equivalent. Additionally, setting a price this high is not particularly unrealistic considering it's the starting price for a dutch auction in which it should be intentionally priced much higher than it's worth.

##### Proof of Concept:

We can create the following test in `AstariaTest.t.sol` to verify:

```
function testCannotLiquidateTooHighInitialAsk() public {
  TestNFT nft = new TestNFT(3);
  vm.label(address(nft), "nft");
  address tokenContract = address(nft);
  uint256 tokenId = uint256(1);
  address publicVault = _createPublicVault({
    strategist: strategistOne,
    delegate: strategistTwo,
    epochLength: 14 days
  });

  _lendToVault(
    Lender({addr: address(1), amountToLend: 50 ether}),
    publicVault
  );

  uint256 vaultTokenBalance = IERC20(publicVault).balanceOf(address(1));
  ILienToken.Stack[] memory stack;
  (, stack) = _commitToLien({
    vault: publicVault,
    strategist: strategistOne,
    strategistPK: strategistOnePK,
    tokenContract: tokenContract,
    tokenId: tokenId,
    lienDetails: ILienToken.Details({
      maxAmount: 50 ether,
      rate: (uint256(1e16) * 150) / (365 days),
      duration: 10 days,
      maxPotentialDebt: 0 ether,
      liquidationInitialAsk: type(uint256).max
    }),
    amount: 5 ether,
    isFirstLien: true
  });

  uint256 collateralId = tokenContract.computeId(tokenId);

  skip(14 days); // end of loan
  vm.expectRevert();
  OrderParameters memory listedOrder = ASTARIA_ROUTER.liquidate(
    stack,
    uint8(0)
  );
}
```

##### Remediation:

This can be avoided by using a uint256 as the `auctionData.startAmount`.