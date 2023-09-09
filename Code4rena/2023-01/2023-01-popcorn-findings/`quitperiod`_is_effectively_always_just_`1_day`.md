## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-02

# [`quitPeriod` is effectively always just `1 day`](https://github.com/code-423n4/2023-01-popcorn-findings/issues/785) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L629-L636


# Vulnerability details

## Description
The `quitPeriod` is supposed to give users time to rage quit if there are changes they don't agree with. The quit period is limited to be within 1 day and a week and can only be changed by `owner`:

```javascript
File: Vault.sol

629:    function setQuitPeriod(uint256 _quitPeriod) external onlyOwner {
630:        if (_quitPeriod < 1 days || _quitPeriod > 7 days)
631:            revert InvalidQuitPeriod();
632:
633:        quitPeriod = _quitPeriod;
634:
635:        emit QuitPeriodSet(quitPeriod);
636:    }
```

However the change can be done instantly. An owner can propose a change, users will expect to wait three days for it to be applied, and after one day change the `quitPeriod` to `1 day` and apply the changes.

## Impact
Changes to fees and adapters can happen faster than users expect not giving them time enough to react.

## Proof of Concept
Small PoC in `Vault.t.sol`:

```javascript
  function test__set_fees_after_1_day() public {
    vault.proposeFees(
      VaultFees({
        deposit: 1e17,
        withdrawal: 1e17,
        management: 1e17,
        performance: 1e17
      })
    );
    // users expect to have three days
    console.log("quit period",vault.quitPeriod());

    // jump 1 day
    vm.warp(block.timestamp + 1 days);
    // owner changes quit period
    vault.setQuitPeriod(1 days);
    // and does the changes
    vault.changeFees();
  }
```

## Tools Used
manual audit, vs code, forge

## Recommended Mitigation Steps
Either lock `quitPeriod` changes for the old `quitPeriod`.

Or apply the duration when the change is proposed:
```diff
diff --git a/src/vault/Vault.sol b/src/vault/Vault.sol
index 7a8f941..bccc561 100644
--- a/src/vault/Vault.sol
+++ b/src/vault/Vault.sol
@@ -531,14 +531,14 @@ contract Vault is
         ) revert InvalidVaultFees();
 
         proposedFees = newFees;
-        proposedFeeTime = block.timestamp;
+        proposedFeeTime = block.timestamp + quitPeriod;
 
         emit NewFeesProposed(newFees, block.timestamp);
     }
 
     /// @notice Change fees to the previously proposed fees after the quit period has passed.
     function changeFees() external {
-        if (block.timestamp < proposedFeeTime + quitPeriod)
+        if (block.timestamp < proposedFeeTime)
             revert NotPassedQuitPeriod(quitPeriod);
 
         emit ChangedFees(fees, proposedFees);
```

Same applies for `changeAdapter`