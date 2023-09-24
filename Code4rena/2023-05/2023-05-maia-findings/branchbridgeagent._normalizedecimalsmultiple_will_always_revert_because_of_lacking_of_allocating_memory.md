## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-15

# [BranchBridgeAgent._normalizeDecimalsMultiple will always revert because of lacking of allocating memory](https://github.com/code-423n4/2023-05-maia-findings/issues/598) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/BranchBridgeAgent.sol#L1349-L1357


# Vulnerability details

## Impact
BranchBridgeAgent._normalizeDecimalsMultiple will always revert because of lacking of allocating memory

## Proof of Concept
[BranchBridgeAgent._normalizeDecimalsMultiple](https://github.com/code-423n4/2023-05-maia/blob/54a45beb1428d85999da3f721f923cbf36ee3d35/src/ulysses-omnichain/BranchBridgeAgent.sol#L1349-L1357)'s code as below, because `deposits` is never allocated memory, the function will always revert
```solidity
    function _normalizeDecimalsMultiple(uint256[] memory _deposits, address[] memory _tokens)
        internal
        view
        returns (uint256[] memory deposits)
    {
        for (uint256 i = 0; i < _deposits.length; i++) {
            deposits[i] = _normalizeDecimals(_deposits[i], ERC20(_tokens[i]).decimals());
        }
    }
```

## Tools Used
VS
## Recommended Mitigation Steps
```solidity
@@ -1351,7 +1351,9 @@
         view
         returns (uint256[] memory deposits)
     {
-        for (uint256 i = 0; i < _deposits.length; i++) {
+        uint len = _deposits.length;
+        deposits = new uint256[](len);
+        for (uint256 i = 0; i < len; i++) {
             deposits[i] = _normalizeDecimals(_deposits[i], ERC20(_tokens[i]).decimals());
         }
     }

```


## Assessed type

Error