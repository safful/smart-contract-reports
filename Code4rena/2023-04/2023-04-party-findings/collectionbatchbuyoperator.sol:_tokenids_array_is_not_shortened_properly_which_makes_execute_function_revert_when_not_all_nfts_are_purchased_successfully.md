## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- edited-by-warden
- M-11

# [CollectionBatchBuyOperator.sol: tokenIds array is not shortened properly which makes execute function revert when not all NFTs are purchased successfully](https://github.com/code-423n4/2023-04-party-findings/issues/4) 

# Lines of code

https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/operators/CollectionBatchBuyOperator.sol#L180-L183


# Vulnerability details

## Impact
The [`CollectionBatchBuyOperator`](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/operators/CollectionBatchBuyOperator.sol#L14-L224) contract allows parties to buy NFTs through proposals.  

The proposal specifies an `nftContract` and token IDs (via the `nftTokenIdsMerkleRoot` parameter) that can be bought.  

Allowed `executors` can then execute the actual purchase by executing the proposal and providing execution data.  

The execution data specifies which token IDs to buy, where to buy them from and the price to buy the tokens for.  

The `CollectionBatchBuyOperator.execute` function is supposed to succeed even when not all purchases are successful.  

This is achieved by skipping over failed purchases:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/operators/CollectionBatchBuyOperator.sol#L132-L137)  
```solidity
{
    // Execute the call to buy the NFT.
    (bool success, ) = _buy(call.target, callValue, call.data);


    if (!success) continue;
}
```

Later in the function the NFTs that have been bought are transferred to the party:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/operators/CollectionBatchBuyOperator.sol#L186-L188)  
```solidity
for (uint256 i; i < tokenIds.length; ++i) {
    op.nftContract.safeTransferFrom(address(this), msg.sender, tokenIds[i]);
}
```

If at least one NFT purchase has failed, the `tokenIds` array is bigger than the amount of NFTs that has actually been purchased. In other words there are empty spots at the end of the `tokenIds` array, i.e. the value that is stored there is zero.  

Therefore, before transferring the NFTs, the `tokenIds` array needs to be shortened such that it is not attempted to transfer `tokenId=0`.  

The contract uses the following code to achieve this:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/operators/CollectionBatchBuyOperator.sol#L180-L183)  
```solidity
assembly {
    // Update length of `tokenIds`
    mstore(mload(ex), tokensBought)
}
```

This code is wrong as I will explain later.  

The impact of this is that when not all purchases are successful the function reverts because it attempts to transfer the `tokenId=0` (since there are empty spots in the `tokenIds` array and the array is not shortened).  

So the execution of the proposal will fail when it should actually succeed.  

## Proof of Concept
Let's have a look again at the code to shorten the `tokenIds` array:  

[Link](https://github.com/code-423n4/2023-04-party/blob/440aafacb0f15d037594cebc85fd471729bcb6d9/contracts/operators/CollectionBatchBuyOperator.sol#L180-L183)  
```solidity
assembly {
    // Update length of `tokenIds`
    mstore(mload(ex), tokensBought)
}
```

It loads the first 32 bytes of `ex` from memory (`ex` is a `CollectionBatchBuyExecutionData` struct) and stores `tokensBought` in the memory location where the 32 bytes point to.  

This has nothing to do with shortening the `tokenIds` array.  

The correct code would be:  

```solidity
assembly {
    // Update length of `tokenIds`
    mstore(tokenIds, tokensBought)
}
```

This writes `tokensBought` to the first 32 bytes slot of the `tokenIds` array which is where the size of the array is stored.  

There exists a test case for this scenario in the `CollectionBatchBuyOperator.t.sol` test file. However the test contains an error which makes the test pass even though the `tokenIds` array is not shortened.  

Apply these changes to the test file:  
```diff
diff --git a/sol-tests/operators/CollectionBatchBuyOperator.t.sol b/sol-tests/operators/CollectionBatchBuyOperator.t.sol
index 3956e84..a944c74 100644
--- a/sol-tests/operators/CollectionBatchBuyOperator.t.sol
+++ b/sol-tests/operators/CollectionBatchBuyOperator.t.sol
@@ -165,7 +165,7 @@ contract CollectionBatchBuyOperatorTest is Test, TestUtils, ERC721Receiver {
         bytes memory executionData = abi.encode(
             CollectionBatchBuyOperator.CollectionBatchBuyExecutionData({
                 calls: calls,
-                numOfTokens: 2
+                numOfTokens: 3
             })
         );
```

Notice that when running the test (with the changes to the test file applied) it fails since the `tokenIds` array is not shortened properly.  

Then also apply the changes to the source file (shortening the array properly) and see that the test passes.  

## Tools Used
VSCode, Foundry

## Recommended Mitigation Steps
As explained above, this is how to properly shorten the `tokenIds` array:  

```diff
diff --git a/contracts/operators/CollectionBatchBuyOperator.sol b/contracts/operators/CollectionBatchBuyOperator.sol
index 4b1dcc9..fffa5e9 100644
--- a/contracts/operators/CollectionBatchBuyOperator.sol
+++ b/contracts/operators/CollectionBatchBuyOperator.sol
@@ -179,7 +179,7 @@ contract CollectionBatchBuyOperator is IOperator {
 
         assembly {
             // Update length of `tokenIds`
-            mstore(mload(ex), tokensBought)
+            mstore(tokenIds, tokensBought)
         }
```


