## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-03

# [`excuteProposal` can fail due to Wormhole guardian change](https://github.com/code-423n4/2023-07-moonwell-findings/issues/325) 

# Lines of code

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/Governance/TemporalGovernor.sol#L346-L350


# Vulnerability details

## Impact
Wormhole governance can change signing guardian sets. If this happens between a proposal is queued and a proposal is executed. The second verification in `_executeProposal` will fail as the guardian set has changed between queuing and executing.

This would cause a proposal to fail to execute after the `proposalDelay` has passed. Causing a lengthy re-sending of the message from `Timelock` on source chain, then `proposalDelay` again.

## Proof of Concept
To execute a proposal on `TemporalGovernor` it first needs to be queued. When queued it is checked that the message is valid against the Wormhole bridge contract:

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/Governance/TemporalGovernor.sol#L295-L306
```solidity
File: Governance/TemporalGovernor.sol

295:    function _queueProposal(bytes memory VAA) private {
296:        /// Checks
297:
298:        // This call accepts single VAAs and headless VAAs
299:        (
300:            IWormhole.VM memory vm,
301:            bool valid,
302:            string memory reason
303:        ) = wormholeBridge.parseAndVerifyVM(VAA);
304:
305:        // Ensure VAA parsing verification succeeded.
306:        require(valid, reason);
```

After some more checks are done the message is [queued by its wormhole hash](https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/Governance/TemporalGovernor.sol#L338-L339).

Then, after `proposalDelay`, anyone can call `executeProposal`

Before the proposal is executed though, a second check against the Wormhole bridge contract is done:

https://github.com/code-423n4/2023-07-moonwell/blob/main/src/core/Governance/TemporalGovernor.sol#L344-L350
```solidity
File: Governance/TemporalGovernor.sol

344:    function _executeProposal(bytes memory VAA, bool overrideDelay) private {
345:        // This call accepts single VAAs and headless VAAs
346:        (
347:            IWormhole.VM memory vm,
348:            bool valid,
349:            string memory reason
350:        ) = wormholeBridge.parseAndVerifyVM(VAA);
```

The issue is that the second time the verification might fail.

During the wormhole contract check for validity there is a check that the correct guardian set has signed the message:

https://github.com/wormhole-foundation/wormhole/blob/f11299b888892d5b4abaa624737d0e5117d1bbb2/ethereum/contracts/Messages.sol#L79-L82
```solidity
79:        /// @dev Checks if VM guardian set index matches the current index (unless the current set is expired).
80:        if(vm.guardianSetIndex != getCurrentGuardianSetIndex() && guardianSet.expirationTime < block.timestamp){
81:            return (false, "guardian set has expired");
82:        }
```

But(!), Wormhole governance can change the signers:

https://github.com/wormhole-foundation/wormhole/blob/f11299b888892d5b4abaa624737d0e5117d1bbb2/ethereum/contracts/Governance.sol#L76-L112
```solidity
 76:    /**
 77:     * @dev Deploys a new `guardianSet` via Governance VAA/VM
 78:     */
 79:    function submitNewGuardianSet(bytes memory _vm) public {
            // ... parsing and verification

104:        // Trigger a time-based expiry of current guardianSet
105:        expireGuardianSet(getCurrentGuardianSetIndex());
106:
107:        // Add the new guardianSet to guardianSets
108:        storeGuardianSet(upgrade.newGuardianSet, upgrade.newGuardianSetIndex);
109:
110:        // Makes the new guardianSet effective
111:        updateGuardianSetIndex(upgrade.newGuardianSetIndex);
112:    }
```

Were this to happen between `queueProposal` and `executeProposal` the proposal would fail to execute.

## Tools Used
Manual audit

## Recommended Mitigation Steps
Consider only check the `VAA`s validity against Wormhole in `_executeProposal` when it is fasttracked (`overrideDelay==true`).

However then anyone can just just take the hash from the proposal `VAA` and submit whatever commands. One idea to prevent this is to instead of storing the `VAA` `vm.hash`, use the hash of the complete `VAA` as key in `queuedTransactions`. Thus, the content cannot be altered.

Or, the whole `VAA` can be stored alongside the timestamp and the `executeProposal` call can be made with just the hash.


## Assessed type

Timing