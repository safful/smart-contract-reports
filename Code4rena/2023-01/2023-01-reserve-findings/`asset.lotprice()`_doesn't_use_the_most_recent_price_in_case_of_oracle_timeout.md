## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- M-08

# [`Asset.lotPrice()` doesn't use the most recent price in case of oracle timeout](https://github.com/code-423n4/2023-01-reserve-findings/issues/326) 

# Lines of code

https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/plugins/assets/Asset.sol#L144-L145


# Vulnerability details

## Impact

`Asset.lotPrice()` has a fallback mechanism in case that `tryPrice()` fails - it uses the last saved price and multiplies its value by `lotMultiplier` (a variable that decreases as the time since the last saved price increase) and returns the results.
However, the `tryPrice()` might fail due to oracle timeout, in that case the last saved price might be older than the oracle's price.

This can cause the backing manager to misestimate the value of the asset, trade it at a lower price, or do an unnecessary haircut.

## Proof of Concept
In the PoC below:
* Oracle price is set at day 0
* The asset is refreshed (e.g. somebody issued/vested/redeemed)
* After 5 days the oracle gets an update
* 25 hours later the `lotPrice()` is calculated based on the oracle price from day 0 even though a price from day 5 is available from the oracle
* Oracle gets another update
* 25 hours later the `lotPrice()` goes down to zero since it considers the price from day 0 (which is more than a week ago) to be the last saved price, even though a price from a day ago is available from the oracle

```diff
diff --git a/test/fixtures.ts b/test/fixtures.ts
index 5299a5f6..75ca8010 100644
--- a/test/fixtures.ts
+++ b/test/fixtures.ts
@@ -69,7 +69,7 @@ export const SLOW = !!useEnv('SLOW')
 
 export const PRICE_TIMEOUT = bn('604800') // 1 week
 
-export const ORACLE_TIMEOUT = bn('281474976710655').div(2) // type(uint48).max / 2
+export const ORACLE_TIMEOUT = bn('86400') // one day
 
 export const ORACLE_ERROR = fp('0.01') // 1% oracle error
 
diff --git a/test/plugins/Asset.test.ts b/test/plugins/Asset.test.ts
index d49c53f3..7f2f721e 100644
--- a/test/plugins/Asset.test.ts
+++ b/test/plugins/Asset.test.ts
@@ -233,6 +233,45 @@ describe('Assets contracts #fast', () => {
       )
     })
 
+    it('PoC lot price doesn\'t use most recent price', async () => {
+      // Update values in Oracles to 0
+
+      await setOraclePrice(rsrAsset.address, bn('1.1e8'))
+
+      await rsrAsset.refresh();
+      let [lotLow, lotHigh] = await rsrAsset.lotPrice();
+      let descripion = "day 0";
+      console.log({descripion, lotLow, lotHigh});
+      let hour = 60*60;
+      let day = hour*24;
+      await advanceTime(day * 5);
+
+      await setOraclePrice(rsrAsset.address, bn('2e8'));
+      // await rsrAsset.refresh();
+
+      [lotLow, lotHigh] = await rsrAsset.lotPrice();
+      descripion = 'after 5 days (right after update)';
+      console.log({descripion,lotLow, lotHigh});
+
+      await advanceTime(day + hour);
+
+      // Fallback prices should be zero
+
+      [lotLow, lotHigh] = await rsrAsset.lotPrice();
+      descripion = 'after 6+ days';
+      console.log({descripion, lotLow, lotHigh});
+
+      await setOraclePrice(rsrAsset.address, bn('2e8'));
+
+      await advanceTime(day + hour);
+
+      [lotLow, lotHigh] = await rsrAsset.lotPrice();
+      descripion = 'after 7+ days';
+      console.log({descripion, lotLow, lotHigh});
+
+    })
+    return;
+
     it('Should return (0, 0) if price is zero', async () => {
       // Update values in Oracles to 0
       await setOraclePrice(compAsset.address, bn('0'))
@@ -595,6 +634,7 @@ describe('Assets contracts #fast', () => {
       expect(lotHighPrice4).to.be.equal(bn(0))
     })
   })
+  return;
 
   describe('Constructor validation', () => {
     it('Should not allow price timeout to be zero', async () => {

```

Output:
```
{
  descripion: 'day 0',
  lotLow: BigNumber { value: "1089000000000000000" },
  lotHigh: BigNumber { value: "1111000000000000000" }
}
{
  descripion: 'after 5 days (right after update)',
  lotLow: BigNumber { value: "1980000000000000000" },
  lotHigh: BigNumber { value: "2020000000000000000" }
}
{
  descripion: 'after 6+ days',
  lotLow: BigNumber { value: "149087485119047618" },
  lotHigh: BigNumber { value: "152099353505291005" }
}
{
  descripion: 'after 7+ days', // `lotPrice()` returns zero even though the most recent price the oracle holds is from 25 hours ago
  lotLow: BigNumber { value: "0" },
  lotHigh: BigNumber { value: "0" }
}
```

## Recommended Mitigation Steps
Allow specifying a timeout to `tryPrice()`, in case that `tryPrice()` fails due to oracle timeout then call it again with `priceTimeout` as the timeout.
If the call succeeds the second time then use it as the most recent price for fallback calculations. 