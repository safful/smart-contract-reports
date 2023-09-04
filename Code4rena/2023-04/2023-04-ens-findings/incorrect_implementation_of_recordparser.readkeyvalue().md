## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-04

# [Incorrect implementation of RecordParser.readKeyValue()](https://github.com/code-423n4/2023-04-ens-findings/issues/246) 

# Lines of code

https://github.com/code-423n4/2023-04-ens/blob/45ea10bacb2a398e14d711fe28d1738271cd7640/contracts/dnsregistrar/RecordParser.sol#L34


# Vulnerability details

## Impact
`RecordParser.readKeyValue()` returns a wrong `value` if the terminator not found.
This is a fundamental library and any contract using it may experience unexpected errors and problems due to this bug.

## Proof of Concept
The implementation logic of `RecordParser.readKeyValue(bytes memory input, uint256 offset, uint256 len)` is roughly as follows:
1. Find the character `=` in the range `offset..(offset+len)` of input and record its position in `separator`.
2. Find the space character in the range `(separator+1)...(offset+len)` of input, and record its position in `terminator`.
3. Return `key: input[offset..separator]`, `value: input[(separator+1)..terminator]`, `nextOffset: terminator+1`

The problem is that if the space is not found in step 2, terminator will be set to `input.length` - [RecordParser.sol#L34](https://github.com/code-423n4/2023-04-ens/blob/45ea10bacb2a398e14d711fe28d1738271cd7640/contracts/dnsregistrar/RecordParser.sol#L34):
```
    if (terminator == type(uint256).max) {
        terminator = input.length;
    }
```
This is incorrect because the parameters passed require data to be read within range `offset..(offset+len)`, it should not return a value beyond `offset+len`.

For example, suppose we have: `input = "...;key1=val1 key2=val2;..."` and `offset` is the start position of `key2`.

If we call `readKeyValue(input, offset, 9)`, the function will return:
```
key: "key2"
value: "val2;..."
nextOffset: input.length+1
```

The returned `value` is wrong due to the incorrect implementation of RecordParser.

The correct return should be:
```
key: "key2"
value: "val2"
nextOffset: offset+9
```

Test code for PoC:
```
diff --git a/contracts/dnsregistrar/mocks/DummyParser.sol b/contracts/dnsregistrar/mocks/DummyParser.sol
new file mode 100644
index 0000000..538e652
--- /dev/null
+++ b/contracts/dnsregistrar/mocks/DummyParser.sol
@@ -0,0 +1,34 @@
+pragma solidity ^0.8.4;
+
+import "../../dnssec-oracle/BytesUtils.sol";
+import "../RecordParser.sol";
+
+contract DummyParser {
+    using BytesUtils for bytes;
+
+    // parse data in format: name;key1=value1 key2=value2;url
+    function parseData(
+        bytes memory data,
+        uint256 kvCount
+    ) external pure returns (string memory name, string[] memory keys, string[] memory values, string memory url) {
+        uint256 len = data.length;
+        // retrieve name
+        uint256 sep1 = data.find(0, len, ";");
+        name = string(data.substring(0, sep1));
+
+        // retrieve url
+        uint256 sep2 = data.find(sep1 + 1, len - sep1, ";");
+        url = string(data.substring(sep2 + 1, len - sep2 - 1));
+
+        keys = new string[](kvCount);
+        values = new string[](kvCount);
+        // retrieve keys and values
+        uint256 offset = sep1 + 1;
+        for (uint256 i; i < kvCount && offset < len; i++) {
+            (bytes memory key, bytes memory val, uint256 nextOffset) = RecordParser.readKeyValue(data, offset, sep2 - offset);
+            keys[i] = string(key);
+            values[i] = string(val);
+            offset = nextOffset;
+        }
+    }
+}
diff --git a/test/DummyParser.test.js b/test/DummyParser.test.js
new file mode 100644
index 0000000..396557d
--- /dev/null
+++ b/test/DummyParser.test.js
@@ -0,0 +1,27 @@
+const { expect } = require('chai')
+const { ethers } = require('hardhat')
+const { toUtf8Bytes } = require('ethers/lib/utils')
+
+describe('DummyParser', () => {
+  let parser
+
+  before(async () => {
+    const factory = await ethers.getContractFactory('DummyParser')
+    parser = await factory.deploy()
+  })
+
+  it('parse data', async () => {
+    const data = "usdt;issuer=tether decimals=18;https://tether.to"
+    const [name, keys, values, url] = await parser.parseData(toUtf8Bytes(data), 2)
+    // correct name
+    expect(name).to.eq('usdt')
+    // correct keys and values
+    expect(keys[0]).to.eq('issuer')
+    expect(values[0]).to.eq('tether')
+    expect(keys[1]).to.eq('decimals')
+    // incorrect last value
+    expect(values[1]).to.eq('18;https://tether.to')
+    // correct url
+    expect(url).to.eq('https://tether.to')
+  })
+})
\ No newline at end of file
```

## Tools Used
VS Code

## Recommended Mitigation Steps

We should change [the assignment of `terminator`](https://github.com/code-423n4/2023-04-ens/blob/45ea10bacb2a398e14d711fe28d1738271cd7640/contracts/dnsregistrar/RecordParser.sol#L34) so that it cannot exceed the query range:

```
diff --git a/contracts/dnsregistrar/RecordParser.sol b/contracts/dnsregistrar/RecordParser.sol
index 339f213..876a87f 100644
--- a/contracts/dnsregistrar/RecordParser.sol
+++ b/contracts/dnsregistrar/RecordParser.sol
@@ -31,11 +31,12 @@ library RecordParser {
             " "
         );
         if (terminator == type(uint256).max) {
-            terminator = input.length;
+            terminator = len + offset;
+            nextOffset = terminator;
+        } else {
+            nextOffset = terminator + 1;
         }
-
         key = input.substring(offset, separator - offset);
         value = input.substring(separator + 1, terminator - separator - 1);
-        nextOffset = terminator + 1;
     }
 }
```
