## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-01

# [HexUtils.hexStringToBytes32() and HexUtils.hexToAddress() may return incorrect results](https://github.com/code-423n4/2023-04-ens-findings/issues/281) 

# Lines of code

https://github.com/code-423n4/2023-04-ens/blob/45ea10bacb2a398e14d711fe28d1738271cd7640/contracts/utils/HexUtils.sol#L11
https://github.com/code-423n4/2023-04-ens/blob/45ea10bacb2a398e14d711fe28d1738271cd7640/contracts/utils/HexUtils.sol#L68


# Vulnerability details

## Impact
The `HexUtils.hexStringToBytes32()` and `HexUtils.hexToAddress()` may return incorrect results if the input data provided is not in a standard format.

This could cause the contract to behave abnormally in some scenarios or be exploited for malicious purposes.

## Proof of Concept
The function `HexUtils.hexStringToBytes32(bytes memory str, uint256 idx, uint256 lastIdx)` is used to convert a hexadecimal string `str[idx...lastIndx]` to a `bytes32`.

However, the function lacks some critical checks on the input data, resulting in the following situations:
1. If the length `lastIdx - idx` is odd, it will not revert, but will read an additional out-of-range byte `str[lastIdx]` and return it.
2. If the length `lstIdx - idx > 32`, it will not revert, but will discard the excess data at the beginning and return the last 32 bytes.
3. If the length `lstIdx - idx < 32`, it will not revert, but will pad the data with zeros at the beginning.

The following test code verifies these situations:

```
diff --git a/test/utils/HexUtils.js b/test/utils/HexUtils.js
index 296eadf..e12e11c 100644
--- a/test/utils/HexUtils.js
+++ b/test/utils/HexUtils.js
@@ -16,6 +16,44 @@ describe('HexUtils', () => {
     HexUtils = await HexUtilsFactory.deploy()
   })

+  describe.only('Special cases for hexStringToBytes32()', () => {
+    const hex32Bytes = '5cee339e13375638553bdf5a6e36ba80fb9f6a4f0783680884d92b558aa471da'
+    it('odd length 1', async () => {
+      let [bytes32, valid] = await HexUtils.hexStringToBytes32(
+        toUtf8Bytes(hex32Bytes), 0, 63,
+      )
+      expect(valid).to.equal(true)
+      // the last 4 bits (half byte) of hex32Bytes is out of range but read
+      expect(bytes32).to.equal('0x' + hex32Bytes)
+    })
+    it('odd length 2', async () => {
+      let [bytes32, valid] = await HexUtils.hexStringToBytes32(
+        toUtf8Bytes(hex32Bytes + '00'), 1, 64,
+      )
+      expect(valid).to.equal(true)
+      // the first half byte of '00' is out of range but read
+      expect(bytes32).to.equal('0x' + hex32Bytes.substring(1) + '0')
+    })
+    it('not enough length', async () => {
+      let [bytes32, valid] = await HexUtils.hexStringToBytes32(
+        toUtf8Bytes(hex32Bytes), 0, 2,
+      )
+      expect(valid).to.equal(true)
+      // only one byte is read, but it is expanded to 32 bytes
+      expect(bytes32).to.equal(
+        '0x000000000000000000000000000000000000000000000000000000000000005c',
+      )
+    })
+    it('exceed length', async () => {
+      let [bytes32, valid] = await HexUtils.hexStringToBytes32(
+        toUtf8Bytes(hex32Bytes + "1234"), 0, 64 + 4,
+      )
+      expect(valid).to.equal(true)
+      // 34 bytes is read, and returns the last 32 bytes
+      expect(bytes32).to.equal('0x' + hex32Bytes.substring(4) + '1234')
+    })
+  })
+
   describe('hexStringToBytes32()', () => {
     it('Converts a hex string to bytes32', async () => {
       let [bytes32, valid] = await HexUtils.hexStringToBytes32(
```

Test code outputs:
```
  HexUtils
    Special cases for hexStringToBytes32()
      ✓ odd length 1
      ✓ odd length 2
      ✓ not enough length
      ✓ exceed length

```

Since [HexUtils.hexToAddress()](https://github.com/code-423n4/2023-04-ens/blob/45ea10bacb2a398e14d711fe28d1738271cd7640/contracts/utils/HexUtils.sol#L68) is implemented by directly calling `HexUtils.hexStringToBytes32()`, it also has similar issues.


## Tools Used
VS Code

## Recommended Mitigation Steps
Should revert the function if the input length `lastIdx - idx` is odd.

For cases where the length is greater than or less than 32 (or 20)
- if the current implementation meets the requirements, the design should be detailed in a comment
- otherwise the function should revert if the length is not 32 (or 20)
