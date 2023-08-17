## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- H-06

# [Buying out corrupts the slope of a vault, reducing rewards of LPs](https://github.com/code-423n4/2023-01-astaria-findings/issues/477) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L189
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L627-L628


# Vulnerability details

## Impact
After a buyout, the slope of a vault won't be increased. As a result, liquidity providers will lose reward for providing liquidity to borrowers and the borrower will not pay interest for the lien that was bought out.
## Proof of Concept
Buyout is an important refinancing mechanism that allows borrowers to apply new terms (e.g. changed loan rate and/our duration) to their loans. [The implementation of the mechanism](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/VaultImplementation.sol#L313) allows borrower to repay the owed amount for a lien, burn the lien, and create a new lien. When burning and creating liens it's important to update [the slope of a vault](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L627-L628): is the total interest accrued by vaults. However, during a buyout the slope of the vault where a new lien is created is not increased:
1. after a new lien is created, the slope of the vault is not increased ([LienToken.sol#L127](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L127));
1. however, the slope of the vault is decreased after the old lien is burned ([LienToken.sol#L189](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L189), [PublicVault.sol#L627-L628](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L627-L628))

Since, during a buyout, a lien with a different interest rate may be created (due to changed loan terms), the slope of the vault must be updated correctly:
1. the slope of the previous lien must be reduced from the total slope of the vault;
1. the slope of the new lien must be added to the total slope of the vault.

If the slope of the new lien is not added to the total slope of the vault, then the lien doesn't generate interest, which means the borrower doesn't need to pay interest for taking the loan and liquidity providers won't be rewarded for providing funds to the borrower.

The following PoC demonstrates that the slope of a vault is 0 after the only existing lien was bought out:
```solidity
// src/test/AstariaTest.t.sol
function testBuyoutLienWrongSlope_AUDIT() public {
  TestNFT nft = new TestNFT(1);
  address tokenContract = address(nft);
  uint256 tokenId = uint256(0);

  uint256 initialBalance = WETH9.balanceOf(address(this));

  // create a PublicVault with a 14-day epoch
  address publicVault = _createPublicVault({
    strategist: strategistOne,
    delegate: strategistTwo,
    epochLength: 14 days
  });
  vm.label(publicVault, "PublicVault");

  // lend 50 ether to the PublicVault as address(1)
  _lendToVault(
    Lender({addr: address(1), amountToLend: 50 ether}),
    publicVault
  );

  // borrow 10 eth against the dummy NFT
  (uint256[] memory liens, ILienToken.Stack[] memory stack) = _commitToLien({
    vault: publicVault,
    strategist: strategistOne,
    strategistPK: strategistOnePK,
    tokenContract: tokenContract,
    tokenId: tokenId,
    lienDetails: standardLienDetails,
    amount: 10 ether,
    isFirstLien: true
  });

  // Right after the lien was created the slope of the vault equals to the slope of the lien.
  assertEq(PublicVault(publicVault).getSlope(), LIEN_TOKEN.calculateSlope(stack[0]));

  vm.warp(block.timestamp + 3 days);

  IAstariaRouter.Commitment memory refinanceTerms = _generateValidTerms({
    vault: publicVault,
    strategist: strategistOne,
    strategistPK: strategistOnePK,
    tokenContract: tokenContract,
    tokenId: tokenId,
    lienDetails: refinanceLienDetails,
    amount: 10 ether,
    stack: stack
  });

  (stack, ) = VaultImplementation(publicVault).buyoutLien(
    stack,
    uint8(0),
    refinanceTerms
  );

  // After a buyout the slope of the vault is 0, however it must be equal to the slope of the lien.
  // Error: a == b not satisfied [uint]
  //  Expected: 481511019363
  //  Actual: 0
  assertEq(PublicVault(publicVault).getSlope(), LIEN_TOKEN.calculateSlope(stack[0]));

  // A lien exists after the buyout, so the slope of the vault cannot be 0.
  uint256 collId = stack[0].lien.collateralId;
  assertEq(LIEN_TOKEN.getCollateralState(collId), bytes32(hex"7c2c35af3fb5f00ff3995cdddd95dcbbad96eea7bca39b305f2d0f8a55d838f8"));
}
```
## Tools Used
Manual review
## Recommended Mitigation Steps
Consider increasing the slope of a public vault after a buyout, similarly to how it's done [after a new commitment](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L449).