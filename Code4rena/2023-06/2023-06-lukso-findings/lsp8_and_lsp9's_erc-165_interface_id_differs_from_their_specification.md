## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-03

# [LSP8 and LSP9's ERC-165 interface ID differs from their specification](https://github.com/code-423n4/2023-06-lukso-findings/issues/122) 

# Lines of code

https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP7DigitalAsset/LSP7Constants.sol#L4-L5
https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP8IdentifiableDigitalAsset/LSP8Constants.sol#L4-L5


# Vulnerability details

## Bug Description

According to [LSP7's specification](https://github.com/lukso-network/LIPs/blob/main/LSPs/LSP-7-DigitalAsset.md#specification), the [ERC-165](https://eips.ethereum.org/EIPS/eip-165) interface ID for LSP7 token contracts should be `0x5fcaac27`:

> ERC165 interface id: `0x5fcaac27`

However, `_INTERFACEID_LSP7` has a different value in the code:

[LSP7Constants.sol#L4-L5](https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP7DigitalAsset/LSP7Constants.sol#L4-L5)

```solidity
// --- ERC165 interface ids
bytes4 constant _INTERFACEID_LSP7 = 0xda1f85e4;
```

Similarly, LSP8's interface ID should be `0x49399145` according to [LSP8's specification](https://github.com/lukso-network/LIPs/blob/main/LSPs/LSP-8-IdentifiableDigitalAsset.md#specification):

> ERC165 interface id: `0x49399145`

However, `_INTERFACEID_LSP8` has a different value in the code:

[LSP8Constants.sol#L4-L5](https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP8IdentifiableDigitalAsset/LSP8Constants.sol#L4-L5)

```solidity
// --- ERC165 interface ids
bytes4 constant _INTERFACEID_LSP8 = 0x622e7a01;
```

These constants are used in `supportsInterface()` for the `LSP7DigitalAsset` and `LSP8IdentifiableDigitalAsset` contracts.

## Impact

Protocols that check for LSP7/LSP8 compatibility using the ERC-165 interface IDs declared in the specification will receive incorrect return values when calling `supportsInterface()`.

## Recommended Mitigation

Ensure that the interface ID declared in the code matches their respective ones in their specifications.


## Assessed type

Error