## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-04

# [`LSP8Burnable` extension incorrectly inherits `LSP8IdentifiableDigitalAssetCore`](https://github.com/code-423n4/2023-06-lukso-findings/issues/120) 

# Lines of code

https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP8IdentifiableDigitalAsset/extensions/LSP8Burnable.sol#L15


# Vulnerability details

## Bug Description

The `LSP8Burnable` contract inherits from `LSP8IdentifiableDigitalAssetCore`:

[LSP8Burnable.sol#L15](https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP8IdentifiableDigitalAsset/extensions/LSP8Burnable.sol#L15)

```solidity
abstract contract LSP8Burnable is LSP8IdentifiableDigitalAssetCore {
```

However, LSP8 extensions are supposed to inherit `LSP8IdentifiableDigitalAsset` instead. This can be inferred by looking at [`LSP8CappedSupply.sol`](https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP8IdentifiableDigitalAsset/extensions/LSP8CappedSupply.sol), [`LSP8CompatibleERC721.sol`](https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP8IdentifiableDigitalAsset/extensions/LSP8CompatibleERC721.sol) and [`LSP8Enumerable.sol`](https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP8IdentifiableDigitalAsset/extensions/LSP8Enumerable.sol):

[LSP8CappedSupply.sol#L13](https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP8IdentifiableDigitalAsset/extensions/LSP8CappedSupply.sol#L13)

```solidity
abstract contract LSP8CappedSupply is LSP8IdentifiableDigitalAsset {
```

Additionally, the `LSP8BurnableInitAbstract.sol` file is missing in the repository.

## Impact

As `LSP8Burnable` does not inherit `LSP8IdentifiableDigitalAsset`, a developer who implements his LSP8 token using `LSP8Burnable` will face the following issues:

* All functionality from `LSP4DigitalAssetMetadata` will be unavailable.
* As `LSP8Burnable` does not contain a `supportsInterface()` function, it will be incompatible with contracts that use [ERC-165](https://eips.ethereum.org/EIPS/eip-165).

## Recommended Mitigation

The `LSP8Burnable` contract should inherit `LSP8IdentifiableDigitalAsset` instead:

[LSP8Burnable.sol#L15](https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP8IdentifiableDigitalAsset/extensions/LSP8Burnable.sol#L15)

```diff
-   abstract contract LSP8Burnable is LSP8IdentifiableDigitalAssetCore {
+   abstract contract LSP8Burnable is LSP8IdentifiableDigitalAsset {
```

Secondly, add a `LSP8BurnableInitAbstract.sol` file that contains an implementation of `LSP8Burnable` which can be used in proxies.


## Assessed type

Other