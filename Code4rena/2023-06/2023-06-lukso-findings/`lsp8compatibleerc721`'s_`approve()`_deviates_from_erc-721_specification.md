## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-05

# [`LSP8CompatibleERC721`'s `approve()` deviates from ERC-721 specification](https://github.com/code-423n4/2023-06-lukso-findings/issues/118) 

# Lines of code

https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP8IdentifiableDigitalAsset/extensions/LSP8CompatibleERC721.sol#L155-L158


# Vulnerability details

## Bug Description

The `LSP8CompatibleERC721` contract is a wrapper around LSP8 that is meant to function similarly to ERC-721 tokens. One of its implemented functions is ERC-721's `approve()`:

[LSP8CompatibleERC721.sol#L155-L158](https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP8IdentifiableDigitalAsset/extensions/LSP8CompatibleERC721.sol#L155-L158)

```solidity
    function approve(address operator, uint256 tokenId) public virtual {
        authorizeOperator(operator, bytes32(tokenId));
        emit Approval(tokenOwnerOf(bytes32(tokenId)), operator, tokenId);
    }
```

As `approve()` calls `authorizeOperator()` from the `LSP8IdentifiableDigitalAssetCore` contract, only the owner of `tokenId` is allowed to call `approve()`:

[LSP8IdentifiableDigitalAssetCore.sol#L105-L113](https://github.com/code-423n4/2023-06-lukso/blob/main/contracts/LSP8IdentifiableDigitalAsset/LSP8IdentifiableDigitalAssetCore.sol#L105-L113)

```solidity
    function authorizeOperator(
        address operator,
        bytes32 tokenId
    ) public virtual {
        address tokenOwner = tokenOwnerOf(tokenId);

        if (tokenOwner != msg.sender) {
            revert LSP8NotTokenOwner(tokenOwner, tokenId, msg.sender);
        }
```

However, the implementation above deviates from the [ERC-721 specification](https://eips.ethereum.org/EIPS/eip-721), which mentions that an "authorized operator of the current owner" should also be able to call `approve()`:

```solidity
    /// @notice Change or reaffirm the approved address for an NFT
    /// @dev The zero address indicates there is no approved address.
    ///  Throws unless `msg.sender` is the current NFT owner, or an authorized
    ///  operator of the current owner.
    /// @param _approved The new approved NFT controller
    /// @param _tokenId The NFT to approve
    function approve(address _approved, uint256 _tokenId) external payable;
```

This means that anyone who is an approved operator for `tokenId`'s owner through `setApprovalForAll()` should also be able to grant approvals. An example of such behaviour can be seen in [Openzeppelin's ERC721 implementation](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/ERC721.sol):

[ERC721.sol#L121-L123](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/ERC721.sol#L121-L123)

```solidity
        if (_msgSender() != owner && !isApprovedForAll(owner, _msgSender())) {
            revert ERC721InvalidApprover(_msgSender());
        }
```

## Impact

As `LSP8CompatibleERC721`'s `approve()` functions differently from ERC-721, protocols that rely on this functionality will be incompatible with LSP8 tokens that inherit from `LSP8CompatibleERC721`. 

For example, in an NFT exchange, users might be required to call `setApprovalForAll()` for the protocol's router contract. The router then approves a swap contract, which transfers the NFT from the user to the recipient using `transferFrom()`.

Additionally, developers that expect `LSP8CompatibleERC721` to behave exactly like ERC-721 tokens might introduce bugs in their contracts due to the difference in `approve()`.  

## Recommended Mitigation

Modify `approve()` to allow approved operators for `tokenId`'s owner to grant approvals:

```solidity
function approve(address operator, uint256 tokenId) public virtual { 
    bytes32 tokenIdBytes = bytes32(tokenId);
    address tokenOwner = tokenOwnerOf(tokenIdBytes);

    if (tokenOwner != msg.sender && !isApprovedForAll(tokenOwner, msg.sender)) {
        revert LSP8NotTokenOwner(tokenOwner, tokenIdBytes, msg.sender);
    }

    if (operator == address(0)) {
        revert LSP8CannotUseAddressZeroAsOperator();
    }

    if (tokenOwner == operator) {
        revert LSP8TokenOwnerCannotBeOperator();
    }

    bool isAdded = _operators[tokenIdBytes].add(operator);
    if (!isAdded) revert LSP8OperatorAlreadyAuthorized(operator, tokenIdBytes);

    emit AuthorizedOperator(operator, tokenOwner, tokenIdBytes);
    emit Approval(tokenOwner, operator, tokenId);
}
```





## Assessed type

ERC721