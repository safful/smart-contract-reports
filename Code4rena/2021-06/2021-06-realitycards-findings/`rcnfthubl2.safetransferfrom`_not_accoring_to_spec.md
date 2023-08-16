## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed
- resolved

# [`RCNftHubL2.safeTransferFrom` not accoring to spec](https://github.com/code-423n4/2021-06-realitycards-findings/issues/160) 

# Handle

cmichel


# Vulnerability details

## Vulnerability Details

The `RCNftHubL2.safeTransferFrom` function does not correctly implement the ERC721 spec:

> When using safeTransferFrom, the token contract checks to see that the receiver is an IERC721Receiver, which implies that it knows how to handle ERC721 tokens. [ERC721](https://docs.openzeppelin.com/contracts/2.x/api/token/erc721#IERC721-safeTransferFrom)

This check is not implemented, it just drops the `_data` argument.

## Impact

Contracts that don't know how to handle ERC721 tokens (are not an `IERC721Receiver`) can accept them but they should not when using `safeTransferFrom` according to spec.

## Recommended Mitigation Steps

Implement the `IERC721Receiver` check in `safeTransferFrom`.


