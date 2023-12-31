## Tags

- bug
- duplicate
- 3 (High Risk)
- sponsor confirmed
- selected for report
- responded

# [An attacker can lock operator out of the pod by setting gas limit that's higher than the block gas limit of dest chain](https://github.com/code-423n4/2022-10-holograph-findings/issues/414) 

# Lines of code

https://github.com/code-423n4/2022-10-holograph/blob/f8c2eae866280a1acfdc8a8352401ed031be1373/contracts/HolographOperator.sol#L415


# Vulnerability details


When a beaming job is executed, there's a requirement that the gas left would be at least as the `gasLimit` set by the user.
Given that there's no limit on the `gasLimit` the user can set, a user can set the `gasLimit` to amount that's higher than the block gas limit on the dest chain, causing the operator to fail to execute the job.

## Impact
Operators would be locked out of the pod, unable to execute any more jobs and not being able to get back the bond they paid.

The attacker would have to pay a value equivalent to the gas fee if that amount was realistic (i.e. `gasPrice` * `gasLimit` in dest chain native token), but this can be a relative low amount for Polygon and Avalanche chain (for Polygon that's 20M gas limit and 200 Gwei gas = 4 Matic, for Avalanche the block gas limit seems to be 8M and the price ~30 nAVAX = 0.24 AVAX ). Plus, the operator isn't going to receive that amount.

## Proof of Concept
The following test demonstrates this scenario:

```diff
diff --git a/test/06_cross-chain_minting_tests_l1_l2.ts b/test/06_cross-chain_minting_tests_l1_l2.ts
index 1f2b959..a1a23b7 100644
--- a/test/06_cross-chain_minting_tests_l1_l2.ts
+++ b/test/06_cross-chain_minting_tests_l1_l2.ts
@@ -276,6 +276,7 @@ describe('Testing cross-chain minting (L1 & L2)', async function () {
             gasLimit: TESTGASLIMIT,
           })
         );
+        estimatedGas = BigNumber.from(50_000_000);
         // process.stdout.write('\n' + 'gas estimation: ' + estimatedGas.toNumber() + '\n');
 
         let payload: BytesLike = await l1.bridge.callStatic.getBridgeOutRequestPayload(
@@ -303,7 +304,8 @@ describe('Testing cross-chain minting (L1 & L2)', async function () {
             '0x' + remove0x((await l1.operator.getMessagingModule()).toLowerCase()).repeat(2),
             payload
           );
-
+        estimatedGas = BigNumber.from(5_000_000);
+        
         process.stdout.write(' '.repeat(10) + 'expected lz gas to be ' + executeJobGas(payload, true).toString());
         await expect(
           adminCall(l2.mockLZEndpoint.connect(l2.lzEndpoint), l2.lzModule, 'lzReceive', [
@@ -313,7 +315,7 @@ describe('Testing cross-chain minting (L1 & L2)', async function () {
             payload,
             {
               gasPrice: GASPRICE,
-              gasLimit: executeJobGas(payload),
+              gasLimit: 5_000_000,
             },
           ])
         )
```

The test would fail with the following output:

```
  1) Testing cross-chain minting (L1 & L2)
       Deploy cross-chain contracts via bridge deploy
         hToken
           deploy l1 equivalent on l2:
     VM Exception while processing transaction: revert HOLOGRAPH: not enough gas left
```

## Recommended Mitigation Steps
Limit the `gasLimit` to the maximum realistic amount that can be used on the dest chain (including the gas used up to the point where it's checked).