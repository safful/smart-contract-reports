## Tags

- bug
- 2 (Med Risk)
- sponsor confirmed

# [Malicious pausing the contract](https://github.com/code-423n4/2022-09-nouns-builder-findings/issues/719) 

# Lines of code

https://github.com/code-423n4/2022-09-nouns-builder/blob/7e9fddbbacdd7d7812e912a369cfd862ee67dc03/src/auction/Auction.sol#L204
https://github.com/code-423n4/2022-09-nouns-builder/blob/7e9fddbbacdd7d7812e912a369cfd862ee67dc03/src/auction/Auction.sol#L206
https://github.com/code-423n4/2022-09-nouns-builder/blob/7e9fddbbacdd7d7812e912a369cfd862ee67dc03/src/auction/Auction.sol#L235


# Vulnerability details

# Vulnerability details

## Description

There is a function `_createAuction` in `Auction` contract.

It consist the following logic:

```
/// @dev Creates an auction for the next token
function _createAuction() private {
    // Get the next token available for bidding
    try token.mint() returns (uint256 tokenId) {
        **creating of the auction for token with id equal to tokenId**

        // Pause the contract if token minting failed
    } catch Error(string memory) {
        _pause();
    }
}
```

According to the [EIP-150](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-150.md) `call` opcode can consume as most `63/64` of parrent calls' gas. That means `token.mint()` can fail since there will be no gas. 

All in all, if `token.mint()` fail on gas and the rest gas is enough for pausing the contract by calling `_pause` in `catch` statement the contract will be paused.

Please note, that a bug can be exploitable if the token.mint() consume more than 1.500.000 of gas, because 1.500.000 / 64 > 20.000 that need to pause the contract. Also, the logic of `token.mint()` includes traversing the array up to 100 times, that's heavy enough to reach 1.500.000 gas limit. 

## Impact

Contract can be paused by any user by passing special amount of gas for the call of `settleCurrentAndCreateNewAuction` (which consists of two internal calls of `_settleAuction` and `_createAuction` functions).

## Recommended Mitigation Steps

Add a special check for upper bound of `gasLeft` at start of `_createAuction` function.