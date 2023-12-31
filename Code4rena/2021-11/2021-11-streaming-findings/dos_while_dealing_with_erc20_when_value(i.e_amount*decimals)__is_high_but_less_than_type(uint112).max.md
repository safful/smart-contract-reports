## Tags

- bug
- 3 (High Risk)
- sponsor confirmed

# [DOS while dealing with erc20 when value(i.e amount*decimals)  is high but less than type(uint112).max](https://github.com/code-423n4/2021-11-streaming-findings/issues/228) 

# Handle

hack3r-0m


# Vulnerability details

## Impact

https://github.com/code-423n4/2021-11-streaming/blob/main/Streaming/src/Locke.sol#L229

reverts due to overflow for higher values (but strictly less than type(uint112).max) and hence when user calls `exit` or `withdraw` function it will revert and that user will not able to withdraw funds permanentaly.

## Proof of Concept

Attaching diff to modify tests to reproduce behaviour:

```
diff --git a/Streaming/src/test/Locke.t.sol b/Streaming/src/test/Locke.t.sol
index 2be8db0..aba19ce 100644
--- a/Streaming/src/test/Locke.t.sol
+++ b/Streaming/src/test/Locke.t.sol
@@ -166,14 +166,14 @@ contract StreamTest is LockeTest {
         );
 
         testTokenA.approve(address(stream), type(uint256).max);
-        stream.fundStream((10**14)*10**18);
+        stream.fundStream(1000);
 
-        alice.doStake(stream, address(testTokenB), (10**13)*10**18);
+        alice.doStake(stream, address(testTokenB), 100);
 
 
         hevm.warp(startTime + minStreamDuration / 2); // move to half done
         
-        bob.doStake(stream, address(testTokenB), (10**13)*10**18);
+        bob.doStake(stream, address(testTokenB), 100);
 
         hevm.warp(startTime + minStreamDuration / 2 + minStreamDuration / 10);
 
@@ -182,10 +182,10 @@ contract StreamTest is LockeTest {
         hevm.warp(startTime + minStreamDuration + 1); // warp to end of stream
 
 
-        // alice.doClaimReward(stream);
-        // assertEq(testTokenA.balanceOf(address(alice)), 533*(10**15));
-        // bob.doClaimReward(stream);
-        // assertEq(testTokenA.balanceOf(address(bob)), 466*(10**15));
+        alice.doClaimReward(stream);
+        assertEq(testTokenA.balanceOf(address(alice)), 533);
+        bob.doClaimReward(stream);
+        assertEq(testTokenA.balanceOf(address(bob)), 466);
     }
 
     function test_stake() public {
diff --git a/Streaming/src/test/utils/LockeTest.sol b/Streaming/src/test/utils/LockeTest.sol
index eb38060..a479875 100644
--- a/Streaming/src/test/utils/LockeTest.sol
+++ b/Streaming/src/test/utils/LockeTest.sol
@@ -90,11 +90,11 @@ abstract contract LockeTest is TestHelpers {
         testTokenA = ERC20(address(new TestToken("Test Token A", "TTA", 18)));
         testTokenB = ERC20(address(new TestToken("Test Token B", "TTB", 18)));
         testTokenC = ERC20(address(new TestToken("Test Token C", "TTC", 18)));
-        write_balanceOf_ts(address(testTokenA), address(this), (10**14)*10**18);
-        write_balanceOf_ts(address(testTokenB), address(this), (10**14)*10**18);
-        write_balanceOf_ts(address(testTokenC), address(this), (10**14)*10**18);
-        assertEq(testTokenA.balanceOf(address(this)), (10**14)*10**18);
-        assertEq(testTokenB.balanceOf(address(this)), (10**14)*10**18);
+        write_balanceOf_ts(address(testTokenA), address(this), 100*10**18);
+        write_balanceOf_ts(address(testTokenB), address(this), 100*10**18);
+        write_balanceOf_ts(address(testTokenC), address(this), 100*10**18);
+        assertEq(testTokenA.balanceOf(address(this)), 100*10**18);
+        assertEq(testTokenB.balanceOf(address(this)), 100*10**18);
 
         defaultStreamFactory = new StreamFactory(address(this), address(this));
 
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

Consider doing arithmetic operations in two steps or upcasting to u256 and then downcasting. Alternatively, find a threshold where it breaks and add require condition to not allow total stake per user greater than threshhold.

