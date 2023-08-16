## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Caching a storage load in TridentERC20](https://github.com/code-423n4/2021-09-sushitrident-findings/issues/116) 

# Handle

hrkrshnn


# Vulnerability details

## Caching a storage load in TridentERC20

[Context](https://github.com/sushiswap/trident/blob/9130b10efaf9c653d74dc7a65bde788ec4b354b5/contracts/pool/TridentERC20.sol#L76)

This avoids an unnecessary `sload`.

``` diff
modified   contracts/pool/TridentERC20.sol
@@ -73,8 +73,9 @@ abstract contract TridentERC20 {
         address recipient,
         uint256 amount
     ) external returns (bool) {
-        if (allowance[sender][msg.sender] != type(uint256).max) {
-            allowance[sender][msg.sender] -= amount;
+        uint _allowance = allowance[sender][msg.sender];
+        if (_allowance != type(uint256).max) {
+            allowance[sender][msg.sender] = _allowance - amount;
         }
         balanceOf[sender] -= amount;
         // @dev This is safe from overflow - the sum of all user
```


