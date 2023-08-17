## Tags

- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- MR-H-08

# [Attacker can steal the NFT bought by sending it to another vault he control](https://github.com/code-423n4/2023-01-tessera-mitigation-findings/issues/47) 

# Lines of code

https://github.com/fractional-company/modular-fractional/blob/d0974eef922c3087ed9c1b8d758fe66307734c23/src/modules/GroupBuy.sol#L214


# Vulnerability details

## Impact

The mitigation of H-08 try to validate the vault returned by _market with the VaultRegistry. However, it only validated if the vault exists, but not if it is the correct vault. A similar attack described in https://github.com/code-423n4/2022-12-tessera-findings/issues/47 can be carried out by using a valid vault address that is permissionlessly deployed with VaultRegistry.createFor but have the owner set to the attacker.

## Proof of Concept
https://github.com/fractional-company/modular-fractional/blob/d0974eef922c3087ed9c1b8d758fe66307734c23/src/modules/GroupBuy.sol#L214-L215
```solidity
        (, uint256 id) = IVaultRegistry(registry).vaultToToken(vault);
        if (id == 0) revert NotVault();
```

1. Group assembles and raises funds to buy NFT X
2. Attacker deploy a valid vault using VaultRegistry.createFor but setting himself as the owner, with a merkle root that enable NFT transfer
3. Attacker calls purchase() and supplies their malicious contract in _market, returning the vault generated above
4. Attacker call Vault.execute in the vault to retrieve the NFT to his own wallet
5. Attacker receives the NFT bought by the raised funds with 0 cost except gas

https://github.com/fractional-company/modular-fractional/blob/9411afcd03d457192adc2bc59b3b378aeddd5865/src/Vault.sol#L36
```solidity
    function execute(
        address _target,
        bytes calldata _data,
        bytes32[] calldata _proof
    ) external payable returns (bool success, bytes memory response) {
        bytes4 selector;
        assembly {
            selector := calldataload(_data.offset)
        }

        // Generate leaf node by hashing module, target and function selector.
        bytes32 leaf = keccak256(abi.encode(msg.sender, _target, selector));
        // Check that the caller is either a module with permission to call or the owner.
        if (!MerkleProof.verify(_proof, MERKLE_ROOT(), leaf)) {
            if (msg.sender != FACTORY() && msg.sender != OWNER())
                revert NotAuthorized(msg.sender, _target, selector);
        }

        (success, response) = _execute(_target, _data);
    }
```

## Recommended Mitigation Steps
Also check the owner of the vault