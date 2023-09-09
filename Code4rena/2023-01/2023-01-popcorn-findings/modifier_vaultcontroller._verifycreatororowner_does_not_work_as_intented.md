## Tags

- bug
- 3 (High Risk)
- disagree with severity
- primary issue
- selected for report
- sponsor confirmed
- edited-by-warden
- H-12

# [Modifier VaultController._verifyCreatorOrOwner does not work as intented](https://github.com/code-423n4/2023-01-popcorn-findings/issues/45) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/VaultController.sol#L666-L670
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/VaultController.sol#L448
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/VaultController.sol#L608
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/VaultController.sol#L621
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/VaultController.sol#L634
https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/VaultController.sol#L647


# Vulnerability details

## Impact
Modifier `VaultController._verifyCreatorOrOwner` does not work. Instead of checking the condition `msg.sender is creator OR owner`, it makes `msg.sender is creator AND owner`. This would block access to all created Vaults for creators and the owner (if he did not create them).
Specifically, the following functions in the `VaultController` are affected:
- `addStakingRewardsTokens()`;
- `deployVault()`, which has a call to `addStakingRewardsTokens()`, cannot be executed if the argument `rewardsData.length != 0`;
- `pauseAdapters()`;
- `pauseVaults()`;
- `unpauseAdapters()`;
- `unpauseVaults()`.

## Proof of Concept
To check this concept, we can make a truth table for the main condition in the modifier `msg.sender != metadata.creator || msg.sender != owner`. The table shows that the condition will equal `false` only in the one case where `msg.sender` is both creator and owner.

| msg.sender != metadata.creator | msg.sender != owner | msg.sender != metadata.creator \|\| msg.sender != owner |
| ------------------------------ | ------------------- | ------------------------------------------------------- |
| 0                              | 0                   | 0                                                       |
| 0                              | 1                   | 1                                                       |
| 1                              | 0                   | 1                                                       |
| 1                              | 1                   | 1                                                       |

The correct condition should be the following: `msg.sender != metadata.creator && msg.sender != owner`.

| msg.sender != metadata.creator | msg.sender != owner | msg.sender != metadata.creator && msg.sender != owner |
| ------------------------------ | ------------------- | ----------------------------------------------------- |
| 0                              | 0                   | 0                                                     |
| 0                              | 1                   | 0                                                     |
| 1                              | 0                   | 0                                                     |
| 1                              | 1                   | 1                                                     |

In this case, a revert will only happen when `msg.sender` is neither a creator nor the owner, as it should be according to the documentation.

You can also use the following test to check; add it to the file `test\vault\VaultController.t.sol`:
```solidity
function testFail__deployVault_creator_is_not_owner_audit() public {
    addTemplate("Adapter", templateId, adapterImpl, true, true);
    addTemplate("Strategy", "MockStrategy", strategyImpl, false, true);
    addTemplate("Vault", "V1", vaultImpl, true, true);
    controller.setPerformanceFee(uint256(1000));
    controller.setHarvestCooldown(1 days);
    rewardToken.mint(bob, 10 ether);
    rewardToken.approve(address(controller), 10 ether);

    swapTokenAddresses[0] = address(0x9999);
    address adapterClone = 0xD6C5fA22BBE89db86245e111044a880213b35705;
    address strategyClone = 0xe8a41C57AB0019c403D35e8D54f2921BaE21Ed66;
    address stakingClone = 0xE64C695617819cE724c1d35a37BCcFbF5586F752;

    uint256 callTimestamp = block.timestamp;
    vm.prank(bob);
    address vaultClone = controller.deployVault(
        VaultInitParams({
            asset: iAsset,
            adapter: IERC4626(address(0)),
            fees: VaultFees({
                deposit: 100,
                withdrawal: 200,
                management: 300,
                performance: 400
            }),
            feeRecipient: feeRecipient,
            owner: bob
        }),
        DeploymentArgs({id: templateId, data: abi.encode(uint256(100))}),
        DeploymentArgs({id: "MockStrategy", data: ""}),
        address(0),
        abi.encode(
            address(rewardToken),
            0.1 ether,
            1 ether,
            true,
            10000000,
            2 days,
            1 days
        ),
        VaultMetadata({
            vault: address(0),
            staking: address(0),
            creator: bob,
            metadataCID: metadataCid,
            swapTokenAddresses: swapTokenAddresses,
            swapAddress: address(0x5555),
            exchange: uint256(1)
        }),
        0
    );
}
```
In the test's log (`forge test --match-test "testFail__deployVault_creator_is_not_owner" -vvvv`), you can see that the call ended with revert `NotSubmitterNorOwner(0x000000000000000000000000000000000000000000000000DCbA)`.

## Tools Used
Manual Review, VSCodium, Forge

## Recommended Mitigation Steps
Change the condition to `msg.sender != metadata.creator && msg.sender != owner`.