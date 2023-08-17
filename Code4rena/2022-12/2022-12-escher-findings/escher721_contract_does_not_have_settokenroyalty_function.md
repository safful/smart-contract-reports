## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- M-01

# [Escher721 contract does not have setTokenRoyalty function](https://github.com/code-423n4/2022-12-escher-findings/issues/68) 

# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/5d8be6aa0e8634fdb2f328b99076b0d05fefab73/src/Escher721.sol#L10-L99


# Vulnerability details

## Impact
On the code4rena page of this contest there is a "SALES PATTERNS" section that describes the flow of how to use Sales:  

[https://code4rena.com/contests/2022-12-escher-contest](https://code4rena.com/contests/2022-12-escher-contest)  

It contains this statement:  
> If the artist would like sales and royalties to go somewhere other than the default royalty receiver, they  
> must call setTokenRoyalty with the following variables

So an artist should be able to call the `setTokenRoyalty` function on the `Escher721` contract.  

However this function cannot be called. It does not exist. There exists a `_setTokenRoyalty` in `ERC2981.sol` from openzeppelin. This function however is `internal` ([https://github.com/OpenZeppelin/openzeppelin-contracts/blob/3d7a93876a2e5e1d7fe29b5a0e96e222afdc4cfa/contracts/token/common/ERC2981.sol#L94](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/3d7a93876a2e5e1d7fe29b5a0e96e222afdc4cfa/contracts/token/common/ERC2981.sol#L94)).  

So there is no `setTokenRoyalty` function that can be called by the artist. So an artist cannot set a royalty individually for a token as is stated in the documentation.  

## Proof of Concept
I tried calling `setTokenRoyalty` from inside `Escher721.t.sol` with the following test:  
```solidity
function test_setDefaultRoyalty() public {
    edition.setTokenRoyalty(1,address(this), 5);
}
```
This won't even compile because the `setTokenRoyalty` function does not exist.  

## Tools Used
VSCode

## Recommended Mitigation Steps
In order to expose the internal `_setTokenRoyalty` function to the artist, add the following function to the `Escher721` contract:  

```solidity
function setTokenRoyalty(
    uint256 tokenId,
    address receiver,
    uint96 feeNumerator
) public onlyRole(DEFAULT_ADMIN_ROLE) {
    _setTokenRoyalty(tokenId,receiver,feeNumerator);
}
```