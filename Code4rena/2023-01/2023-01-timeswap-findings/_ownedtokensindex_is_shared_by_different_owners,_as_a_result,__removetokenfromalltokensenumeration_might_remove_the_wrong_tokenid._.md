## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-06

# [_ownedTokensIndex is SHARED by different owners, as a result, _removeTokenFromAllTokensEnumeration might remove the wrong tokenId. ](https://github.com/code-423n4/2023-01-timeswap-findings/issues/168) 

# Lines of code

https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-token/src/base/ERC1155Enumerable.sol#L136-L149


# Vulnerability details

## Impact
Detailed description of the impact of this finding.
The data structure 
``_ownedTokensIndex`` is SHARED by different owners, as a result, ``_removeTokenFromAllTokensEnumeration()`` might remove the wrong tokenId. 

## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.

  
``_ownedTokensIndex`` is used to map from token ID to index of the owner tokens list, unfortunately, all owners share the same data structure at the same time (non-fungible tokens). So, the mapping for one owner might be overwritten by another owner when ``_addTokenToOwnerEnumeration`` is called:
https://github.com/code-423n4/2023-01-timeswap/blob/ef4c84fb8535aad8abd6b67cc45d994337ec4514/packages/v2-token/src/base/ERC1155Enumerable.sol#L116-L121.
As a result,    ``_removeTokenFromOwnerEnumeration()`` might remove the wrong tokenID. 

Removing the wrong tokenID can happen like the following:

1) Suppose Alice owns three tokens A, B, C with indices 1 -> A, 2->B, 3->C
2) Suppose Bob owns token D, 1->D, and will add A to his list via ``_addTokenToOwnerEnumeration()``. As a result, we have 1->D, and 2-A, since ``_ownedTokensIndex`` is shared, we have A->2 in ``_ownedTokensIndex``.
3) Next, ``_removeTokenFromOwnerEnumeration()`` is called to remove A from Alice. However, ``tokenIndex`` will be 2, 
 which points to B, as a result, instead of deleting A, B is deleted from ``_ownedTokens``. Wrong token delete!

```
function _removeTokenFromOwnerEnumeration(address from, uint256 tokenId) private {
        uint256 lastTokenIndex = _currentIndex[from] - 1;
        uint256 tokenIndex = _ownedTokensIndex[tokenId];

        if (tokenIndex != lastTokenIndex) {
            uint256 lastTokenId = _ownedTokens[from][lastTokenIndex];

            _ownedTokens[from][tokenIndex] = lastTokenId;
            _ownedTokensIndex[lastTokenId] = tokenIndex;
        }

        delete _ownedTokensIndex[tokenId];
        delete _ownedTokens[from][lastTokenIndex];
    }
```




## Tools Used
Remix

## Recommended Mitigation Steps
Redefine ``ownedTokensIndex`` so that is is not shared:
```
mapping(address => mapping(uint256 => uint256)) private _ownedTokensIndex;
```