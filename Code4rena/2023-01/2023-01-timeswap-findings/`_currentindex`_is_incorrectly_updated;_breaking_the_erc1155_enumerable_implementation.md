## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-01

# [`_currentIndex` is incorrectly updated; breaking the ERC1155 enumerable implementation](https://github.com/code-423n4/2023-01-timeswap-findings/issues/272) 

# Lines of code

https://github.com/code-423n4/2023-01-timeswap/blob/3be51465583552cce76816a05170fda7da68596a/packages/v2-token/src/base/ERC1155Enumerable.sol#L92-L101
https://github.com/code-423n4/2023-01-timeswap/blob/3be51465583552cce76816a05170fda7da68596a/packages/v2-token/src/base/ERC1155Enumerable.sol#L116-L121
https://github.com/code-423n4/2023-01-timeswap/blob/3be51465583552cce76816a05170fda7da68596a/packages/v2-token/src/base/ERC1155Enumerable.sol#L136-L149


# Vulnerability details

## Impact

When minting and burning tokens,the ERC1155Enumerable implementation does not correctly update the following states:
- uint256[] private _allTokens;
- mapping(uint256 => uint256) private _allTokensIndex;
- mapping(address => uint256) internal _currentIndex;

In particular:
- the _allTokens array length (and therefore the totalSupply()) always increases (never decreases)
- the _allTokensIndex[id] always increases
- the _curentIndex[from] always increases

## Proof of Concept

NOTE: the following test requires some private states of ERC1155Enumerable.sol to be set from private to internal.

```solidity
contract HelperERC1155 is ERC1155Enumerable, ERC1155Holder {

    constructor() ERC1155("Test") {
    }

    function mint(uint256 id, uint256 amount) external {
        _mint(msg.sender, id, amount, bytes(""));
    }

    function burn(uint256 id, uint256 amount) external {
        _burn(msg.sender, id, amount);
    }

    function currentIndex(address owner) external view returns (uint256) {
        return _currentIndex[owner];
    }

    function allTokensIndex(uint256 id) external view returns (uint256) {
        return _allTokensIndex[id];
    }

    function allTokens(uint256 idx) external view returns (uint256) {
        return _allTokens[idx];
    }

    function idTotalSupply(uint256 id) external view returns (uint256) {
        return _idTotalSupply[id];
    }
}

contract BugTest is Test, ERC1155Holder {

    function testImplError() public {
        HelperERC1155 token = new HelperERC1155();

        for(uint i=0; i<10; i++){
            token.mint(i, 1+i);
        }

        for(uint i=0; i<10; i++){
            token.burn(i, 1+i);
            assertEq(token.idTotalSupply(i), 0); // OK
            assertEq(token.allTokensIndex(i), i); // NOT OK (should be 0)
        }
        
        assertEq(token.totalSupply(), 10); // NOT OK (should be 0)
        assertEq(token.currentIndex(address(this)), 10); // NOT OK (should be 0)
    }

    function testImplFixed() public {
        HelperERC1155 token = new HelperERC1155();

        for(uint i=0; i<10; i++){
            token.mint(i, 1+i);
        }

        for(uint i=0; i<10; i++){
            token.burn(i, 1+i);
            assertEq(token.idTotalSupply(i), 0); // OK
            assertEq(token.allTokensIndex(i), 0); // OK
        }
        
        assertEq(token.totalSupply(), 0); // OK
        assertEq(token.currentIndex(address(this)), 0); // OK
    }
}
```

Before fix `forge test --match-contract BugTest -vvv` outputs:

```
Running 2 tests for test/Audit2.t.sol:BugTest
[PASS] testImplError() (gas: 2490610)
[FAIL. Reason: Assertion failed.] testImplFixed() (gas: 2560628)
Test result: FAILED. 1 passed; 1 failed; finished in 2.05ms
```

After fix `forge test --match-contract BugTest -vvv` outputs:

```
Running 2 tests for test/Audit2.t.sol:BugTest
[FAIL. Reason: Assertion failed.] testImplError() (gas: 2558695)
[PASS] testImplFixed() (gas: 2489080)
Test result: FAILED. 1 passed; 1 failed; finished in 2.22ms
```

## Tools Used
n/a

## Recommended Mitigation Steps

Correct the implementation to update states correctly. Patch provided below for reference.

```solidity
diff --git a/packages/v2-token/src/base/ERC1155Enumerable.sol b/packages/v2-token/src/base/ERC1155Enumerable.sol
index 4ec23ff..ef67bca 100644
--- a/packages/v2-token/src/base/ERC1155Enumerable.sol
+++ b/packages/v2-token/src/base/ERC1155Enumerable.sol
@@ -91,8 +91,8 @@ abstract contract ERC1155Enumerable is IERC1155Enumerable, ERC1155 {
     /// @dev Remove token enumeration list if necessary.
     function _removeTokenEnumeration(address from, address to, uint256 id, uint256 amount) internal {
         if (to == address(0)) {
-            if (_idTotalSupply[id] == 0 && _additionalConditionRemoveTokenFromAllTokensEnumeration(id)) _removeTokenFromAllTokensEnumeration(id);
             _idTotalSupply[id] -= amount;
+            if (_idTotalSupply[id] == 0 && _additionalConditionRemoveTokenFromAllTokensEnumeration(id)) _removeTokenFromAllTokensEnumeration(id);
         }

         if (from != address(0) && from != to) {
@@ -114,8 +114,7 @@ abstract contract ERC1155Enumerable is IERC1155Enumerable, ERC1155 {
     /// @param to address representing the new owner of the given token ID
     /// @param tokenId uint256 ID of the token to be added to the tokens list of the given address
     function _addTokenToOwnerEnumeration(address to, uint256 tokenId) private {
-        _currentIndex[to] += 1;
-        uint256 length = _currentIndex[to];
+        uint256 length = _currentIndex[to]++;
         _ownedTokens[to][length] = tokenId;
         _ownedTokensIndex[tokenId] = length;
     }
@@ -134,7 +133,7 @@ abstract contract ERC1155Enumerable is IERC1155Enumerable, ERC1155 {
     /// @param from address representing the previous owner of the given token ID
     /// @param tokenId uint256 ID of the token to be removed from the tokens list of the given address
     function _removeTokenFromOwnerEnumeration(address from, uint256 tokenId) private {
-        uint256 lastTokenIndex = _currentIndex[from] - 1;
+        uint256 lastTokenIndex = --_currentIndex[from];
         uint256 tokenIndex = _ownedTokensIndex[tokenId];

         if (tokenIndex != lastTokenIndex) {
```