## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- sponsor confirmed
- H-08

# [Attacker can deploys vaults with a malicious Staking contract](https://github.com/code-423n4/2023-01-popcorn-findings/issues/275) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/VaultController.sol#L106-L110
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/VaultRegistry.sol#L44-L53


# Vulnerability details

## Impact
Anyone can deploy a Vault with a malicious Staking contract attached. If the Staking contract already exists, we can just pass its address to `deployVault` and no checks will be applied to see whether the Staking contract matches valid Staking templates in the Template Registry. 

An attacker can create malicious Staking contract that acts like a regular ERC-4626 vault but with a backdoor function that allows them to withdraw all the deposited funds in the contract. Users may assume the Staking contract is valid and safe and will deposit their funds into it. This will lead to loss of funds for users and huge loss of credibility for the protocol.


## Proof of Concept
The below PoC shows the behavior described above where any Staking contract can be deployed with a Vault. The below lines will need to be added to the `VaultController.t.sol` file. 

```solidity
  function test__deploy_malicious_staking_contract() public {
    addTemplate("Adapter", templateId, adapterImpl, true, true);
    addTemplate("Strategy", "MockStrategy", strategyImpl, false, true);
    addTemplate("Vault", "V1", vaultImpl, true, true);

    // Pretend this malicious Staking contract allows attacker to withdraw
    // all the funds from it while allowing users to use it like a normal Staking contract
    MultiRewardStaking maliciousStaking = new MultiRewardStaking();

    vm.startPrank(alice);
    address vault = controller.deployVault(
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
      address(maliciousStaking),
      "",
      VaultMetadata({
        vault: address(0),
        staking: address(maliciousStaking),
        creator: alice,
        metadataCID: metadataCid,
        swapTokenAddresses: swapTokenAddresses,
        swapAddress: address(0x5555),
        exchange: uint256(1)
      }),
      0
    );
    vm.stopPrank();

    assertEq(vaultRegistry.getVault(vault).staking, address(maliciousStaking));
  }
```

The test can be run with the following command: `forge test --no-match-contract 'Abstract' --match-test test__deploy_malicious_staking_contract`

## Tools Used
VSCode, Foundry

## Recommended Mitigation Steps
1. Add checks to verify that the Staking contract being used in `deployVault` is a Staking contract that was deployed by the system and uses an approved template: 
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/VaultController.sol#L106-L110