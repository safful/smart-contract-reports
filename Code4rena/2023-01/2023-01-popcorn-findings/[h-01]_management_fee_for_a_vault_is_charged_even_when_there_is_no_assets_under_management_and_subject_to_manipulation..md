## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- sponsor confirmed
- M-15

# [[H-01] Management Fee for a vault is charged even when there is no assets under management and subject to manipulation.](https://github.com/code-423n4/2023-01-popcorn-findings/issues/499) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L429-L439
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L473
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L481


# Vulnerability details

`managementFee` is calculated by `accruedManagementFee` function: 
    - managementFee x (totalAssets x (block.timestamp - feesUpdatedAt))/SECONDS_PER_YEAR
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L429-L439

## Impact 1

Management Fee for a vault is charged even when there is no assets under management.

The `feesUpdatedAt` variable is first assigned at block.timestamp when the vault is initialized:
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L87

The vault could be deployed and initialized without any asset under management at time T. For example 10 years after deployment, a user Alice deposits 100ETH into the vault, the management fee will be calculated from T to block.timestamp (which is 10 years) which is not fair. Alice will be charged immediately all majority of 100ETH as management fee. Further than that, if the totalAssets after a year is significant large, the management fee will be highly overcharged for the last year when no fund was managed.

The vault owner could create vaults, wait for a period of time and trap user to deposit. He then could immediately get user assets by claim the wrongful managemennt fee.

## Proof of Concept

<!-- Put the POC in: test/vault/Vault.t.sol -->


    function test__managementFeeOvercharge() public {
        // Set fee
        _setFees(0, 0, 1e17, 0);

        // 10 years passed
        uint256 timestamp = block.timestamp + 315576000; // 10 years
        vm.warp(timestamp);

        // Alice deposit 100 ether to the vault
        uint256 depositAmount = 100 ether;
        asset.mint(alice, depositAmount);
        vm.startPrank(alice);
        asset.approve(address(vault), depositAmount);
        vault.deposit(depositAmount, alice);
        vm.stopPrank();

        // 1 second pass
        uint256 timestamp1 = block.timestamp + 1;
        vm.warp(timestamp1);

        uint256 expectedFeeInAsset = vault.accruedManagementFee();
        uint256 expectedFeeInShares = vault.convertToShares(expectedFeeInAsset);
        
        // Vault creator call takeManagementAndPerformanceFees to take the management fee
        vault.takeManagementAndPerformanceFees();
        console.log("Total Supply: ", vault.totalSupply());
        console.log("Balance of feeRecipient: ", vault.balanceOf(feeRecipient));

        
        assertEq(vault.totalSupply(), depositAmount + expectedFeeInShares);
        assertEq(vault.balanceOf(feeRecipient), expectedFeeInShares);

        // FeeReccipient withdraw the tokens
        vm.startPrank(feeRecipient);
        vault.redeem(vault.balanceOf(feeRecipient), feeRecipient, feeRecipient);
        // 50 ETH is withdrawn to feeRecipient
        console.log("Asset balance of feeRecipient: ", asset.balanceOf(feeRecipient));
        console.log("Vault total assets: ", vault.totalAssets());
    }

## Impact 2

Management Fee is subject to manipulation because of `feesUpdatedAt` and `totalAssets` are varied by user or vault owner's actions
To get the management fee will be lower. 
- A user who wants to deposit large amount of assets to the vault, he will tend to call `takeManagementAndPerformanceFees` to reset the variable `feesUpdatedAt` to block.timestamp before deposit. 
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L473
- A user who want to withdraw will withdraw before the call of `takeManagementAndPerformanceFees` function.

Vault owner will have the incentive to front run a large withdraw of assets and call `takeManagementAndPerformanceFees` to get higher management fee because `totalAssets()` is still high.

## Proof of Concept

Alice deposit 1000 ETH into the vault. The vault deposit, withdraw and management fees are set to 1e17. 

In the first scenario, before Alice can withdraw, vault creator front-run to call the `takeManagementAndPerformanceFees ` function. Result is that feeReceipient will have 192.21 ETH.

<!-- Put the POC in: test/vault/Vault.t.sol -->

    function test__managementFeeFrontRun() public {
        _setFees(1e17, 1e17, 1e17, 0);

        // Alice deposit 1000 ether to the vault
        uint256 depositAmount = 1000 ether;
        asset.mint(alice, depositAmount);
        vm.startPrank(alice);
        asset.approve(address(vault), depositAmount);
        vault.deposit(depositAmount, alice);
        vm.stopPrank();

        // 7 days pass and Alice want to withdraw her ETH from vault
        uint256 timestamp = block.timestamp + 604800;
        vm.warp(timestamp);

        // Vault creator call takeManagementAndPerformanceFees to take the management fee 
        vault.takeManagementAndPerformanceFees();

        // Alice withdraw her ETH
        vm.startPrank(alice);
        vault.redeem(vault.balanceOf(alice), alice, alice);

        uint256 feeInAssetIfFrontRun = vault.convertToAssets(vault.balanceOf(feeRecipient));
        console.log("feeInAssetIfFrontRun: ", feeInAssetIfFrontRun); // feeReceipient will have 192.21 ETH
    }

In the second scenario, no front-run to call the `takeManagementAndPerformanceFees ` function happens. Result is that feeReceipient will have 190 ETH

<!-- Put the POC in: test/vault/Vault.t.sol -->

    function test__managementFeeNoFrontRun() public {
        _setFees(1e17, 1e17, 1e17, 0);

        // Alice deposit 1000 ether to the vault
        uint256 depositAmount = 1000 ether;
        asset.mint(alice, depositAmount);
        vm.startPrank(alice);
        asset.approve(address(vault), depositAmount);
        vault.deposit(depositAmount, alice);
        vm.stopPrank();

        // 7 days pass and Alice want to withdraw her ETH from vault
        uint256 timestamp = block.timestamp + 604800;
        vm.warp(timestamp);

        // Alice withdraw her ETH
        vm.startPrank(alice);
        vault.redeem(vault.balanceOf(alice), alice, alice);

        // Vault creator call takeManagementAndPerformanceFees to take the management fee 
        vault.takeManagementAndPerformanceFees();

        uint256 feeInAssetIfNoFrontRun = vault.convertToAssets(vault.balanceOf(feeRecipient));
        console.log("feeInAssetIfNoFrontRun: ", feeInAssetIfNoFrontRun); // feeReceipient will have 190 ETH
    }

## Tools Used
- Manual

## Recommended Mitigation Steps: 

`feesUpdatedAt` variable is not updated frequently enough. They are only updated when calling `takeManagementAndPerformanceFees` and `changeAdapter`.
The fee should be calculated and took more frequently in each deposit and withdrawal of assets.