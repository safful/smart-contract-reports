## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- M-12

# [Prohibition to create private pools with the factory NFT](https://github.com/code-423n4/2023-04-caviar-findings/issues/353) 

# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L157
https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L623
https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L514


# Vulnerability details

## Impact
Any [Factory NFTs](https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/Factory.sol#L37) deposited into a Factory-PrivatePool can have all assets in the corresponding PrivatePools stolen by malicious users.

## Proof of Concept
Suppose there are two PrivatePools p1 and p2, `p1.nft = address(Factory)`, and `uint256(p1)` and `uint256(p2)` are deposited into p1.
Malicious users can use [flashloan()](https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L623) to steal all the base tokens in p1 and p2:
1. Call `p1.flashloan()` to borrow the Factory NFT - `uint256(p1)` from p1.
2. In the flashloan callback, call `p1.withdraw()` to withdraw all the base tokens and the factory NFT - `uint256(p2)` from p1.
3. Return `uint256(p1)` to p1.

Suppose there are two PrivatePools p1 and p2, `p1.nft = address(Factory)`, and `uint256(p2)` is deposited into p1.
Malicious users can use [flashloan()](https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L623) to steal all the base tokens and NFTs in p2:
1. Call `p1.flashloan()` to borrow factory NFT - `uint256(p2)` from p1.
2. In the flashloan callback, call `p2.withdraw()` to steal all the base tokens and NFTs in p2.
3. Return `uint256(p2)` to p1.

In addition, malicious users can also steal assets in p2 by:
1. [`p1.buy(uint256(p2))`](https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L211)
2. [`p2.withdraw(...)`](https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L514)
3. [`p1.sell(uint256(p2)`](https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/PrivatePool.sol#L301).

## Tools Used
VS Code

## Recommended Mitigation Steps
To prevent users from misusing the protocol and causing financial losses, we should prohibit the creation of PrivatePools with the Factory NFT:
```
diff --git a/src/PrivatePool.sol b/src/PrivatePool.sol
index 75991e1..14ec386 100644
--- a/src/PrivatePool.sol
+++ b/src/PrivatePool.sol
@@ -171,6 +171,8 @@ contract PrivatePool is ERC721TokenReceiver {
         // check that the fee rate is less than 50%
         if (_feeRate > 5_000) revert FeeRateTooHigh();

+        require(_nft != factory, "Unsupported NFT");
+
         // set the state variables
         baseToken = _baseToken;
         nft = _nft;
```

