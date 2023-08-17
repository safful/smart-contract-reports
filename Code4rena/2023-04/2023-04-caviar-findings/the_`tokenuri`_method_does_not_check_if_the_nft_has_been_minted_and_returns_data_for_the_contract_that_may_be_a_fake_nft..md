## Tags

- bug
- 2 (Med Risk)
- judge review requested
- primary issue
- selected for report
- sponsor confirmed
- M-17

# [The `tokenURI` method does not check if the NFT has been minted and returns data for the contract that may be a fake NFT.](https://github.com/code-423n4/2023-04-caviar-findings/issues/44) 

# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/Factory.sol#L161
https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePoolMetadata.sol#L17


# Vulnerability details

## Impact

- By invoking the [Factory.tokenURI](https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/Factory.sol#L161) method for a maliciously provided NFT id, the returned data may deceive potential users, as the method will return data for a non-existent NFT id that appears to be a genuine PrivatePool. This can lead to a poor user experience or financial loss for users.
- Violation of the [ERC721-Metadata part](https://eips.ethereum.org/EIPS/eip-721) standard

## Proof of Concept

- The [Factory.tokenURI](https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/Factory.sol#L161) and [PrivatePoolMetadata.tokenURI](https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePoolMetadata.sol#L17) methods lack any requirements stating that the provided NFT id must be created. We can also see that in the standard implementation by [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/cf86fd9962701396457e50ab0d6cc78aa29a5ebc/contracts/token/ERC721/ERC721.sol#L94), this check is present:
- [Throws if `_tokenId` is not a valid NFT](https://eips.ethereum.org/EIPS/eip-721)

### Example

1. User creates a fake contract
   A simple example so that the `tokenURI` method does not revert:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract NFT {
    function balanceOf(address) external pure returns (uint256) {
        1;
    }
}

contract NonNFT {
    address public immutable nft;

    address public constant baseToken = address(0);
    uint256 public constant virtualBaseTokenReserves = 1 ether;
    uint256 public constant virtualNftReserves = 1 ether;
    uint256 public constant feeRate = 500;

    constructor() {
        nft = address(new NFT());
    }
}
```

2. User deploy the contract
3. Now, by using `tokenURI()` for the deployed user's address, one can fetch information about a non-existent NFT.

## Tools Used

- Manual review
- Foundry

## Recommended Mitigation Steps

- Throw an error if the NFT id is invalid.
