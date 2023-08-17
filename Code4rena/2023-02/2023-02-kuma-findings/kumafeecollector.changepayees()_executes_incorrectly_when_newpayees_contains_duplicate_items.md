## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-02

# [KUMAFeeCollector.changePayees() executes incorrectly when newPayees contains duplicate items](https://github.com/code-423n4/2023-02-kuma-findings/issues/13) 

# Lines of code

https://github.com/code-423n4/2023-02-kuma/blob/3f3d2269fcb3437a9f00ffdd67b5029487435b95/src/kuma-protocol/KUMAFeeCollector.sol#L174-L188


# Vulnerability details

## Impact
When calling [KUMAFeeCollector.changePayees()](https://github.com/code-423n4/2023-02-kuma/blob/3f3d2269fcb3437a9f00ffdd67b5029487435b95/src/kuma-protocol/KUMAFeeCollector.sol#L152) with duplicate payees in `newPayees`, the call is not reverted and the result state will be incorrect.

## Proof of Concept
Contract [KUMAFeeCollector](https://github.com/code-423n4/2023-02-kuma/blob/3f3d2269fcb3437a9f00ffdd67b5029487435b95/src/kuma-protocol/KUMAFeeCollector.sol#L14) does not support duplicate payees. The transaction will revert when trying to add duplicate payees in [addPayee()](https://github.com/code-423n4/2023-02-kuma/blob/3f3d2269fcb3437a9f00ffdd67b5029487435b95/src/kuma-protocol/KUMAFeeCollector.sol#L77):
```
function addPayee(address payee, uint256 share) external override onlyManager {
    if (_payees.contains(payee)) {
        revert Errors.PAYEE_ALREADY_EXISTS();
    }
    ...
}
```

But, function [KUMAFeeCollector.changePayees()](https://github.com/code-423n4/2023-02-kuma/blob/3f3d2269fcb3437a9f00ffdd67b5029487435b95/src/kuma-protocol/KUMAFeeCollector.sol#L152) forgets this constraint, which allows duplicate payees to be passed in `newPayees`. This will cause the contract to record an incorrect state and not work properly.

For example, if `newPayees` contains duplicate payee A, all of its shares will be added to `_totalShares`, but `_shares[A]` will only record the last one.
As a result, the sum of all recorded `_shares` will less than `_totalShares`.

Test code for PoC:
```
diff --git a/test/kuma-protocol/KUMAFeeCollector.t.sol b/test/kuma-protocol/KUMAFeeCollector.t.sol
index f34d9ff..0b3fe46 100644
--- a/test/kuma-protocol/KUMAFeeCollector.t.sol
+++ b/test/kuma-protocol/KUMAFeeCollector.t.sol
@@ -40,6 +40,39 @@ contract KUMAFeeCollectorTest is BaseSetUp {
         );
     }

+    function test_DuplicatePayees() public {
+        address[] memory newPayees = new address[](4);
+        uint256[] memory newShares = new uint256[](4);
+
+        newPayees[0] = vm.addr(10);
+        newPayees[1] = vm.addr(10);
+        newPayees[2] = vm.addr(11);
+        newPayees[3] = vm.addr(12);
+        newShares[0] = 25;
+        newShares[1] = 25;
+        newShares[2] = 25;
+        newShares[3] = 25;
+
+        _KUMAFeeCollector.changePayees(newPayees, newShares);
+
+        // only 3 payees
+        assertEq(_KUMAFeeCollector.getPayees().length, 3);
+        // newPayees[0] and newPayees[1] are identical and both are added as payees[0]
+        address[] memory payees = _KUMAFeeCollector.getPayees();
+        assertEq(payees[0], newPayees[1]);
+        assertEq(payees[1], newPayees[2]);
+        assertEq(payees[2], newPayees[3]);
+
+        uint256 countedTotalShares = 0;
+        for (uint i; i < payees.length; i++) {
+            countedTotalShares += _KUMAFeeCollector.getShare(payees[i]);
+        }
+        // Counted totalShares is 75 (100 - 25)
+        assertEq(countedTotalShares, 75);
+        // Recorded totalShares is 100
+        assertEq(_KUMAFeeCollector.getTotalShares(), 100);
+    }
+
     function test_initialize() public {
         assertEq(address(_KUMAFeeCollector.getKUMAAddressProvider()), address(_KUMAAddressProvider));
         assertEq(_KUMAFeeCollector.getRiskCategory(), _RISK_CATEGORY);
```

Outputs:
```
forge test -m test_DuplicatePayees
[⠔] Compiling...
No files changed, compilation skipped

Running 1 test for test/kuma-protocol/KUMAFeeCollector.t.sol:KUMAFeeCollectorTest
[PASS] test_DuplicatePayees() (gas: 259689)
Test result: ok. 1 passed; 0 failed; finished in 7.39ms
```

## Tools Used
VS Code

## Recommended Mitigation Steps
[KUMAFeeCollector.changePayees()](https://github.com/code-423n4/2023-02-kuma/blob/3f3d2269fcb3437a9f00ffdd67b5029487435b95/src/kuma-protocol/KUMAFeeCollector.sol#L152) should revert if there are duplicates in `newPayees`:

```
diff --git a/src/kuma-protocol/KUMAFeeCollector.sol b/src/kuma-protocol/KUMAFeeCollector.sol
index 402cf71..1a9d86d 100644
--- a/src/kuma-protocol/KUMAFeeCollector.sol
+++ b/src/kuma-protocol/KUMAFeeCollector.sol
@@ -180,7 +180,9 @@ contract KUMAFeeCollector is IKUMAFeeCollector, UUPSUpgradeable, Initializable {
             }

             address payee = newPayees[i];
-            _payees.add(payee);
+            if (!_payees.add(payee)) {
+                revert Errors.PAYEE_ALREADY_EXISTS();
+            }
             _shares[payee] = newShares[i];
             _totalShares += newShares[i];

```

