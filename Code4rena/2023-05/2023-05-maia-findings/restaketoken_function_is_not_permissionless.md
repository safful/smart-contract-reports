## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-19

# [RestakeToken function is not permissionless](https://github.com/code-423n4/2023-05-maia-findings/issues/534) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/uni-v3-staker/UniswapV3Staker.sol#L340-L348
https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/uni-v3-staker/UniswapV3Staker.sol#L373-L374


# Vulnerability details

One of the project assumptions is that anyone can call the `restakeToken` function on someone else's token after the incentive ends (at the start of the new gauge cycle). 

```solidity
File: src/uni-v3-staker/UniswapV3Staker.sol
365:     function _unstakeToken(IncentiveKey memory key, uint256 tokenId, bool isNotRestake) private {
366:         Deposit storage deposit = deposits[tokenId];
367: 
368:         (uint96 endTime, uint256 stakedDuration) =
369:             IncentiveTime.getEndAndDuration(key.startTime, deposit.stakedTimestamp, block.timestamp);
370: 
371:         address owner = deposit.owner;
372: 
373: @>      // anyone can call restakeToken if the block time is after the end time of the incentive
374: @>      if ((isNotRestake || block.timestamp < endTime) && owner != msg.sender) revert NotCalledByOwner();
		
		...
```

This assumption is broken because everywhere the `_unstakeToken` is called, the `isNotRestake` flag is set to `true`, including the [`restakeToken`](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/uni-v3-staker/UniswapV3Staker.sol#L342) function. Because of that, when the caller is not the `deposit.owner`, the `if` block will evaluate to `true`, and the call will revert with `NotCalledByOwner()` error.

```solidity
File: src/uni-v3-staker/UniswapV3Staker.sol
340:     function restakeToken(uint256 tokenId) external {
341:         IncentiveKey storage incentiveId = stakedIncentiveKey[tokenId];
342: @>      if (incentiveId.startTime != 0) _unstakeToken(incentiveId, tokenId, true);
343: 
344:         (IUniswapV3Pool pool, int24 tickLower, int24 tickUpper, uint128 liquidity) =
345:             NFTPositionInfo.getPositionInfo(factory, nonfungiblePositionManager, tokenId);
346: 
347:         _stakeToken(tokenId, pool, tickLower, tickUpper, liquidity);
348:     }
```

## Impact

*Lower yield for users, broken 3rd party integration, higher gas usage*

The purpose of the `restakeToken` function is to: 
- Enable easier automation - re-staking without the need for manual intervention 
- And aggregation - combining multiple actions into a single operation to increase efficiency and reduce transaction costs. 

This is also the reason why the `UniswapV3Staker` contract inherits from `Multicallable`.

Without the ability to re-stake for someone else, 3rd parties or groups of users won't be able to perform cost and yield efficient batch re-stakes. 

As stated in the [Liquidity Mining section](https://v2-docs.maiadao.io/protocols/Hermes/overview/gauges/uni-v3#liquidity-mining) in the docs - LPs will lose new rewards until they re-stake again. Any delay means fewer rewards -> fewer `bHermes` utility tokens -> lower impact in the ecosystem. It is very unlikely that users will be able to re-stake exactly at 12:00 UTC every Thursday (to maximise yield) without some automation/aggregation.

## Proof of Concept

Since I decided to create a fork test on Arbitrum mainnet, the setup is quite lengthy and is explained in great detail in the following [GitHub Gist](https://gist.github.com/ChmielewskiKamil/8261acbbed37b84d176938aa398f19bd).

Pre-conditions:
- Alice and Bob are users of the protocol. They both have the 1000 DAI / 1000 USDC UniswapV3 Liquidity position minted. 
- The UniswapV3Gauge has weight allocated to it. 
- The BaseV2Minter has queued HERMES rewards for the cycle. 
- Charlie is a 3rd party that agreed to re-stake Alice's token at the start of the next cycle (current incentive end time)

```solidity
function testRestake_RestakeIsNotPermissionless() public {
        vm.startPrank(ALICE);
        // 1.a Alice stakes her NFT (at incentive StartTime)
        nonfungiblePositionManager.safeTransferFrom(ALICE, address(uniswapV3Staker), tokenIdAlice);
        vm.stopPrank();

        vm.startPrank(BOB);
        // 1.b Bob stakes his NFT (at incentive StartTime)
        nonfungiblePositionManager.safeTransferFrom(BOB, address(uniswapV3Staker), tokenIdBob);
        vm.stopPrank();

        vm.warp(block.timestamp + 1 weeks); // 2.a Warp to incentive end time
        gauge.newEpoch();                   // 2.b Queue minter rewards for the next cycle

        vm.startPrank(BOB);
        uniswapV3Staker.restakeToken(tokenIdBob); // 3.a Bob can restake his own token
        vm.stopPrank();

        vm.startPrank(CHARLIE);
        vm.expectRevert(bytes4(keccak256("NotCalledByOwner()")));
@>issue uniswapV3Staker.restakeToken(tokenIdAlice); // 3.b Charlie cannot restake Alice's token
        vm.stopPrank();

        uint256 rewardsBob = uniswapV3Staker.rewards(BOB);
        uint256 rewardsAlice = uniswapV3Staker.rewards(ALICE);

        assertNotEq(rewardsBob, 0, "Bob should have rewards");
        assertEq(rewardsAlice, 0, "Alice should not have rewards");

        console.log("=================");
        console.log("Bob's rewards   : %s", rewardsBob);
        console.log("Alice's rewards : %s", rewardsAlice);
        console.log("=================");
    }
```

When used with `multicall` as it probably would in a real-life scenario, it won't work either. 

Change Charlie's part to:

```solidity
bytes memory functionCall1 = abi.encodeWithSignature(
	"restakeToken(uint256)",
	tokenIdAlice
);
bytes memory functionCall2 = abi.encodeWithSignature(
	"restakeToken(uint256)",
	tokenIdBob
);

bytes[] memory data = new bytes[](2);
data[0] = functionCall1;
data[1] = functionCall2;

vm.startPrank(CHARLIE);
address(uniswapV3Staker).call(abi.encodeWithSignature("multicall(bytes[])", data));
vm.stopPrank();
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

A simple fix is to change the `isNotRestake` flag inside the `restakeToken` function to `false`:

```diff
diff --git a/src/uni-v3-staker/UniswapV3Staker.sol b/src/uni-v3-staker/UniswapV3Staker.sol
index 5970379..d7add32 100644
--- a/src/uni-v3-staker/UniswapV3Staker.sol
+++ b/src/uni-v3-staker/UniswapV3Staker.sol
@@ -339,7 +339,7 @@ contract UniswapV3Staker is IUniswapV3Staker, Multicallable {
 
     function restakeToken(uint256 tokenId) external {
         IncentiveKey storage incentiveId = stakedIncentiveKey[tokenId];
-        if (incentiveId.startTime != 0) _unstakeToken(incentiveId, tokenId, true);
+        if (incentiveId.startTime != 0) _unstakeToken(incentiveId, tokenId, false);
 
         (IUniswapV3Pool pool, int24 tickLower, int24 tickUpper, uint128 liquidity) =
             NFTPositionInfo.getPositionInfo(factory, nonfungiblePositionManager, tokenId);
```

A more complicated fix, which would reduce code complexity in the future, would be to rename the `isNotRestake` flag to `isRestake`.

This way, one level of indirection would be reduced.

```diff
diff --git a/src/uni-v3-staker/UniswapV3Staker.sol b/src/uni-v3-staker/UniswapV3Staker.sol
index 5970379..43ff24c 100644
--- a/src/uni-v3-staker/UniswapV3Staker.sol
+++ b/src/uni-v3-staker/UniswapV3Staker.sol
@@ -354,15 +354,15 @@ contract UniswapV3Staker is IUniswapV3Staker, Multicallable {
     /// @inheritdoc IUniswapV3Staker
     function unstakeToken(uint256 tokenId) external {
         IncentiveKey storage incentiveId = stakedIncentiveKey[tokenId];
-        if (incentiveId.startTime != 0) _unstakeToken(incentiveId, tokenId, true);
+        if (incentiveId.startTime != 0) _unstakeToken(incentiveId, tokenId, false);
     }
 
     /// @inheritdoc IUniswapV3Staker
     function unstakeToken(IncentiveKey memory key, uint256 tokenId) external {
-        _unstakeToken(key, tokenId, true);
+        _unstakeToken(key, tokenId, false);
     }
 
-    function _unstakeToken(IncentiveKey memory key, uint256 tokenId, bool isNotRestake) private {
+    function _unstakeToken(IncentiveKey memory key, uint256 tokenId, bool isRestake) private {
         Deposit storage deposit = deposits[tokenId];
 
         (uint96 endTime, uint256 stakedDuration) =
@@ -371,7 +371,7 @@ contract UniswapV3Staker is IUniswapV3Staker, Multicallable {
         address owner = deposit.owner;
 
         // anyone can call restakeToken if the block time is after the end time of the incentive
-        if ((isNotRestake || block.timestamp < endTime) && owner != msg.sender) revert NotCalledByOwner();
+        if ((isRestake || block.timestamp < endTime) && owner != msg.sender) revert NotCalledByOwner();
```


## Assessed type

Access Control