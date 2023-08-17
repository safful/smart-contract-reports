## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-06

# [For a public vault, minimum deposit requirement that is enforced by `ERC4626Cloned.deposit` function can be bypassed by `ERC4626Cloned.mint` function or vice versa when share price does not equal one](https://github.com/code-423n4/2023-01-astaria-findings/issues/486) 

# Lines of code

https://github.com/AstariaXYZ/astaria-gpl/blob/main/src/ERC4626-Cloned.sol#L19-L36
https://github.com/AstariaXYZ/astaria-gpl/blob/main/src/ERC4626-Cloned.sol#L38-L52


# Vulnerability details

## Impact
The following `ERC4626Cloned.deposit` function has `require(shares > minDepositAmount(), "VALUE_TOO_SMALL")` as the minimum deposit requirement, and the `ERC4626Cloned.mint` function below has `require(assets > minDepositAmount(), "VALUE_TOO_SMALL")` as the minimum deposit requirement. For a public vault, when the share price becomes different than 1, these functions' minimum deposit requirements are no longer the same. For example, given `S` is the `shares` input value for the `ERC4626Cloned.mint` function and `A` equals `ERC4626Cloned.previewMint(S)`, when the share price is bigger than 1 and `A` equals `minDepositAmount() + 1`, such `A` will violate the `ERC4626Cloned.deposit` function's minimum deposit requirement but the corresponding `S` will not violate the `ERC4626Cloned.mint` function's minimum deposit requirement; in this case, the user can just ignore the `ERC4626Cloned.deposit` function and call `ERC4626Cloned.mint` function to become a liquidity provider. Thus, when the public vault's share price is different than 1, the liquidity provider can call the less restrictive function out of the two so the minimum deposit requirement enforced by one of the two functions is not effective at all; this can result in unexpected deposit amounts and degraded filtering of who can participate as a liquidity provider.

https://github.com/AstariaXYZ/astaria-gpl/blob/main/src/ERC4626-Cloned.sol#L19-L36
```solidity
  function deposit(uint256 assets, address receiver)
    public
    virtual
    returns (uint256 shares)
  {
    // Check for rounding error since we round down in previewDeposit.
    require((shares = previewDeposit(assets)) != 0, "ZERO_SHARES");

    require(shares > minDepositAmount(), "VALUE_TOO_SMALL");
    // Need to transfer before minting or ERC777s could reenter.
    ERC20(asset()).safeTransferFrom(msg.sender, address(this), assets);

    _mint(receiver, shares);

    ...
  }
```

https://github.com/AstariaXYZ/astaria-gpl/blob/main/src/ERC4626-Cloned.sol#L38-L52
```solidity
  function mint(
    uint256 shares,
    address receiver
  ) public virtual returns (uint256 assets) {
    assets = previewMint(shares); // No need to check for rounding error, previewMint rounds up.
    require(assets > minDepositAmount(), "VALUE_TOO_SMALL");
    // Need to transfer before minting or ERC777s could reenter.
    ERC20(asset()).safeTransferFrom(msg.sender, address(this), assets);

    _mint(receiver, shares);

    ...
  }
```

## Proof of Concept
Please add the following test in `src\test\AstariaTest.t.sol`. This test will pass to demonstrate the described scenario.

```solidity
  function testMinimumDepositRequirementForPublicVaultThatIsEnforcedByDepositFunctionCanBeBypassedByMintFunctionOfERC4626ClonedContractWhenSharePriceIsNotOne() public {
    uint256 budget = 50 ether;
    address alice = address(1);
    address bob = address(2);
    vm.deal(bob, budget);

    TestNFT nft = new TestNFT(2);
    _mintNoDepositApproveRouter(address(nft), 5);
    address tokenContract = address(nft);
    uint256 tokenId = uint256(0);

    address publicVault = _createPublicVault({
      strategist: strategistOne,
      delegate: strategistTwo,
      epochLength: 14 days
    });

    // after alice deposits 50 ether WETH in publicVault, publicVault's share price becomes 1
    _lendToVault(Lender({addr: alice, amountToLend: budget}), publicVault);

    // the borrower borrows 10 ether WETH from publicVault
    (, ILienToken.Stack[] memory stack1) = _commitToLien({
      vault: publicVault,
      strategist: strategistOne,
      strategistPK: strategistOnePK,
      tokenContract: tokenContract,
      tokenId: tokenId,
      lienDetails: standardLienDetails,
      amount: 10 ether,
      isFirstLien: true
    });
    uint256 collateralId = tokenContract.computeId(tokenId);

    // the borrower repays for the lien after 9 days, and publicVault's share price becomes bigger than 1
    vm.warp(block.timestamp + 9 days);
    _repay(stack1, 0, 100 ether, address(this));

    vm.startPrank(bob);

    // bob owns 50 ether WETH
    WETH9.deposit{value: budget}();
    WETH9.approve(publicVault, budget);

    uint256 assetsIn = 100 gwei + 1;

    // for publicVault at this moment, 99265705739 shares are equivalent to (100 gwei + 1) WETH according to previewMint function
    uint256 sharesIn = IERC4626(publicVault).convertToShares(assetsIn);
    assertEq(sharesIn, 99265705739);
    assertEq(IERC4626(publicVault).previewMint(sharesIn), assetsIn);
    
    // bob is unable to call publicVault's deposit function for depositing (100 gwei + 1) WETH
    vm.expectRevert(bytes("VALUE_TOO_SMALL"));
    IERC4626(publicVault).deposit(assetsIn, bob);

    // bob is also unable to call publicVault's deposit function for depositing a little more than (100 gwei + 1) WETH
    vm.expectRevert(bytes("VALUE_TOO_SMALL"));
    IERC4626(publicVault).deposit(assetsIn + 100, bob);

    // however, bob is able to call publicVault's mint function for minting 99265705739 shares while transferring (100 gwei + 1) WETH to publicVault
    IERC4626(publicVault).mint(sharesIn, bob);

    vm.stopPrank();
  }
```

## Tools Used
VSCode

## Recommended Mitigation Steps
The `ERC4626Cloned.deposit` function can be updated to directly compare the `assets` input to `minDepositAmount()` for the minimum deposit requirement while keeping the `ERC4626Cloned.mint` function as is. Alternatively, the `ERC4626Cloned.mint` function can be updated to directly compare the `shares` input to `minDepositAmount()` for the minimum deposit requirement while keeping the `ERC4626Cloned.deposit` function as is.