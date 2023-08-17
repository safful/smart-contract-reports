## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-03

# [Recovery transaction can be replayed after a cancellation](https://github.com/code-423n4/2023-05-ambire-findings/issues/16) 

# Lines of code

https://github.com/AmbireTech/ambire-common/blob/5c54f8005e90ad481df8e34e85718f3d2bfa2ace/contracts/AmbireAccount.sol#L144


# Vulnerability details

# Recovery transaction can be replayed after a cancellation

The recovery transaction can be replayed after a cancellation of the recovery procedure, reinstating the recovery mechanism.  

## Impact

The Ambire wallet provides a recovery mechanism in which a privilege can recover access to the wallet if they lose their keys. The process contains three parts, all of them considered in the `execute()` function:

1. A transaction including a signature with SIGMODE_RECOVER mode enqueues the transaction to be executed after the defined timelock. This action should include a signature by one of the defined recovery keys to be valid.
1. This can be followed by two paths, the cancellation of the process or the execution of the recovery:
   - If the timelock passes, then anyone can complete the execution of the originally submitted bundle.
   - A signed cancellation can be submitted to abort the recovery process, which clears the state of `scheduledRecoveries`.
   
Since nonces are only incremented when the bundle is executed, the call that triggers the recovery procedure can be replayed as long as the nonce stays the same.

This means that the recovery process can be re-initiated after a cancellation is issued by replaying the original call that initiated the procedure.
   
Note that this also works for cancellations. If the submitted recovery bundle is the same, then a cancellation can be replayed if the recovery process is initiated again while under the same nonce value.

## Proof of Concept

1. Recovery process is initiated using a transaction with `SIGMODE_RECOVER` signature mode. 
2. Procedure is canceled by executing a signed call with `SIGMODE_CANCEL` signature mode. 
3. Recovery can be re-initiated by replaying the transaction from step 1.

## Recommendation

Increment the nonce during a cancellation. This will step the nonce preventing any previous signature from being replayed.

```solidity
...
  if (isCancellation) {
    delete scheduledRecoveries[hash];
+   nonce = currentNonce + 1;
    emit LogRecoveryCancelled(hash, recoveryInfoHash, recoveryKey, block.timestamp);
  } else {
    scheduledRecoveries[hash] = block.timestamp + recoveryInfo.timelock;
    emit LogRecoveryScheduled(hash, recoveryInfoHash, recoveryKey, currentNonce, block.timestamp, txns);
  }
  return;
  ...
```




## Assessed type

Other