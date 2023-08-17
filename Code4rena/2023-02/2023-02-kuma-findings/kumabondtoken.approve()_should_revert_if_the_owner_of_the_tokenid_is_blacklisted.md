## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-01

# [KUMABondToken.approve() should revert if the owner of the tokenId is blacklisted](https://github.com/code-423n4/2023-02-kuma-findings/issues/22) 

# Lines of code

https://github.com/code-423n4/2023-02-kuma/blob/3f3d2269fcb3437a9f00ffdd67b5029487435b95/src/mcag-contracts/KUMABondToken.sol#L143


# Vulnerability details

## Impact
It is still possible for a blacklisted user's bond token to be approved.

## Proof of Concept
[KUMABondToken.approve()](https://github.com/code-423n4/2023-02-kuma/blob/3f3d2269fcb3437a9f00ffdd67b5029487435b95/src/mcag-contracts/KUMABondToken.sol#L143) only checks if `msg.sender` and `to` are not blacklisted. It doesn't check if the owner of the `tokenId` is not blacklisted.

For example, the following scenario allows a blacklisted user's bond token to be approved:
1. User A have a bond token bt1.
2. User A calls `KUMABondToken.setApprovalForAll(B, true)`, and user B can operate on all user A's bond tokens.
3. User A is blacklisted.
4. User B calls `KUMABondToken.approve(C, bt1)` to approve user C to operate on bond token bt1.

## Tools Used
VS Code

## Recommended Mitigation Steps
`KUMABondToken.approve()` should revert if the owner of the tokenId is blacklisted:

```
diff --git a/src/mcag-contracts/KUMABondToken.sol b/src/mcag-contracts/KUMABondToken.sol
index 569a042..906fe7b 100644
--- a/src/mcag-contracts/KUMABondToken.sol
+++ b/src/mcag-contracts/KUMABondToken.sol
@@ -146,6 +146,7 @@ contract KUMABondToken is ERC721, Pausable, IKUMABondToken {
         whenNotPaused
         notBlacklisted(to)
         notBlacklisted(msg.sender)
+        notBlacklisted(ERC721.ownerOf(tokenId))
     {
         address owner = ERC721.ownerOf(tokenId);

```

