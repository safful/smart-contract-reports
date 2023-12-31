## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-02

# [Burning a `ERC1155Enumerable` token doesn't remove it from the enumeration ](https://github.com/code-423n4/2023-01-timeswap-findings/issues/248) 

# Lines of code

https://github.com/code-423n4/2023-01-timeswap/blob/main/packages/v2-token/src/base/ERC1155Enumerable.sol#L81-L101


# Vulnerability details

The `ERC1155Enumerable` base contract used in the `TimeswapV2Token` and `TimeswapV2LiquidityToken` tokens provides a functionality to enumerate all token ids that have been minted in the contract.

The logic to remove the token from the enumeration if the last token is burned is implemented in the `_afterTokenTransfer` hook:

https://github.com/code-423n4/2023-01-timeswap/blob/main/packages/v2-token/src/base/ERC1155Enumerable.sol#L81-L101

```solidity
function _afterTokenTransfer(address, address from, address to, uint256[] memory ids, uint256[] memory amounts, bytes memory) internal virtual override {
    for (uint256 i; i < ids.length; ) {
        if (amounts[i] != 0) _removeTokenEnumeration(from, to, ids[i], amounts[i]);

        unchecked {
            ++i;
        }
    }
}

/// @dev Remove token enumeration list if necessary.
function _removeTokenEnumeration(address from, address to, uint256 id, uint256 amount) internal {
    if (to == address(0)) {
        if (_idTotalSupply[id] == 0 && _additionalConditionRemoveTokenFromAllTokensEnumeration(id)) _removeTokenFromAllTokensEnumeration(id);
        _idTotalSupply[id] -= amount;
    }

    if (from != address(0) && from != to) {
        if (balanceOf(from, id) == 0 && _additionalConditionRemoveTokenFromOwnerEnumeration(from, id)) _removeTokenFromOwnerEnumeration(from, id);
    }
}
```

The `_removeTokenEnumeration` condition to check if the supply is 0 happens before the function decreases the burned amount. This will `_removeTokenFromAllTokensEnumeration` from being called when the last token(s) is(are) burned.

## Impact

The token isn't removed from the enumeration since `_removeTokenFromAllTokensEnumeration` will never be called. This will cause the enumeration to always contain a minted token even though it is burned afterwards. The function `totalSupply` and `tokenByIndex` will report wrong values.

This will also cause the enumeration to contain duplicate values or multiple copies of the same token. If the token is minted again after all tokens were previously burned, the token will be re added to the enumeration. 

## PoC

The following test demonstrates the issue. Alice is minted a token and that token is then burned, the token is still present in the enumeration. The token is minted again, causing the enumeration to contain the token by duplicate.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity =0.8.8;

import "forge-std/Test.sol";
import "../src/base/ERC1155Enumerable.sol";

contract TestERC1155Enumerable is ERC1155Enumerable {
    constructor() ERC1155("") {
    }

    function mint(address to, uint256 id, uint256 amount) external {
        _mint(to, id, amount, "");
    }

    function burn(address from, uint256 id, uint256 amount) external {
        _burn(from, id, amount);
    }
}

contract AuditTest is Test {
    function test_ERC1155Enumerable_BadRemoveFromEnumeration() public {
        TestERC1155Enumerable token = new TestERC1155Enumerable();
        address alice = makeAddr("alice");
        uint256 tokenId = 0;
        uint256 amount = 1;

        token.mint(alice, tokenId, amount);

        // tokenByIndex and totalSupply are ok
        assertEq(token.tokenByIndex(0), tokenId);
        assertEq(token.totalSupply(), 1);

        // now we burn the token
        token.burn(alice, tokenId, amount);

        // tokenByIndex and totalSupply still report previous values
        // tokenByIndex should throw index out of bounds, and supply should return 0
        assertEq(token.tokenByIndex(0), tokenId);
        assertEq(token.totalSupply(), 1);

        // Now we mint it again, this will re-add the token to the enumeration, duplicating it
        token.mint(alice, tokenId, amount);
        assertEq(token.totalSupply(), 2);
        assertEq(token.tokenByIndex(0), tokenId);
        assertEq(token.tokenByIndex(1), tokenId);
    }
}
```

## Recommendation

Decrease the amount before checking if the supply is 0. 

```solidity
function _removeTokenEnumeration(address from, address to, uint256 id, uint256 amount) internal {
    if (to == address(0)) {
        _idTotalSupply[id] -= amount;
        if (_idTotalSupply[id] == 0 && _additionalConditionRemoveTokenFromAllTokensEnumeration(id)) _removeTokenFromAllTokensEnumeration(id);
    }

    if (from != address(0) && from != to) {
        if (balanceOf(from, id) == 0 && _additionalConditionRemoveTokenFromOwnerEnumeration(from, id)) _removeTokenFromOwnerEnumeration(from, id);
    }
}
```
