## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-13

# [ `PositionManager` & `PermitERC721` Failure to comply with the EIP-4494](https://github.com/code-423n4/2023-05-ajna-findings/issues/141) 

# Lines of code

https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/PositionManager.sol#L42
https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/base/PermitERC721.sol#L77
https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/base/PermitERC721.sol#L13
https://github.com/code-423n4/2023-05-ajna/blob/276942bc2f97488d07b887c8edceaaab7a5c3964/ajna-core/src/PositionManager.sol#L57


# Vulnerability details


## Impact
The contract `PositionManager.sol` inherits from `PermitERC721.sol`, but both contracts incorrectly implement the `EIP-4494` standard, which is an important part of the contract. This leads to the following issues:

- `PositionManager` & `PermitERC721` are not `EIP-4494` compliant
- Automatic tools will not be able to determine that this contract has a `permit` for `ERC721`
- Third-party contracts will not be able to determine that this is `EIP-4494`
- Inability to correctly track which `nonces` are currently relevant, leading to the creation of invalid signatures/signatures for the future
- No support for compact signatures


## Proof of Concept
According to the specifications of the standard [EIP-4494](https://eips.ethereum.org/EIPS/eip-4494), the following violations were found:

1. ```EIP-4494``` requires the implementation of `IERC165` and the indication of support for the interface ```0x5604e225```, **which is not implemented**

2. `EIP-4494` requires the presence of the function ```function nonces(uint256 tokenId) external view returns(uint256);``` **which is missing**

3. `EIP-4494` requires the function ```function permit(address spender, uint256 tokenId, uint256 deadline, bytes memory sig) external;```, **which is incorrectly declared** as
```javascript
File: 2023-05-ajna\ajna-core\src\base\PermitERC721.sol

77:     function permit(
78:         address spender_, uint256 tokenId_, uint256 deadline_, uint8 v_, bytes32 r_, bytes32 s_
79:     ) external {

```

## Tools Used
* Manual review
* Foundry
* https://eips.ethereum.org/EIPS/eip-4494

## Recommended Mitigation Steps
* Correct the identified non-compliance issues so that the contracts meet the standard
* Or remove references to the standard and provide a reference to the Uniswap V3 implementation





## Assessed type

Other