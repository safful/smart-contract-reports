## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed

# [Changing engine.nft contract breaks vaults](https://github.com/code-423n4/2021-10-mochi-findings/issues/130) 

# Handle

cmichel


# Vulnerability details

Governance can change the `engine.nft` address which is used by vaults to represent collateralized debt positions (CDP).
When minting a vault using `MochiVault.mint` the address returned ID will be used and overwrite the state of an existing debt position and set its status to `Idle`.

## Impact
Changing the NFT address will allow overwriting existing CDPs.

## Recommended Mitigation Steps
Disallow setting a new NFT address. or ensure that the new NFT's IDs start at the old NFT's IDs.

