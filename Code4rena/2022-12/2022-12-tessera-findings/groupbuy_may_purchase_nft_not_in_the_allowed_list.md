## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- M-01

# [GroupBuy may purchase NFT not in the allowed list](https://github.com/code-423n4/2022-12-tessera-findings/issues/14) 

# Lines of code

https://github.com/code-423n4/2022-12-tessera/blob/f37a11407da2af844bbfe868e1422e3665a5f8e4/src/modules/GroupBuy.sol#L186-L188


# Vulnerability details

## Impact

When `_purchaseProof.length == 0`, GroupBuy.purchase compare the tokenId with the merkleRoot. This allow any tokenId that match the merkleRoot to be purchased, even if they are not included in the allow list during setup.

https://github.com/code-423n4/2022-12-tessera/blob/f37a11407da2af844bbfe868e1422e3665a5f8e4/src/modules/GroupBuy.sol#L186-L188

```solidity
        if (_purchaseProof.length == 0) {
            // Hashes tokenId to verify merkle root if proof is empty
            if (bytes32(_tokenId) != merkleRoot) revert InvalidProof();
```

## Proof of Concept

Add the following to GroupBuy.t.sol, it would still revert (since no such nft existed) but not as expected.

```
    // modified from testPurchaseRevertInvalidProof
    function testPurchaseRevertInvalidTokenIdZeroLength() public {
        // setup
        testContributeSuccess();
        // exploit
        uint256 invalidPunkId = uint256(nftMerkleRoot);
        bytes32[] memory invalidPurchaseProof = new bytes32[](0);
        // expect
        vm.expectRevert(INVALID_PROOF_ERROR);
        // execute
        _purchase(
            address(this),
            currentId,
            address(punksBuyer),
            address(punks),
            invalidPunkId,
            minValue,
            purchaseOrder,
            invalidPurchaseProof
        );
    }
```

## Tools Used

Foundry

## Recommended Mitigation Steps

Hash the tokenId even if there is only when length is 1