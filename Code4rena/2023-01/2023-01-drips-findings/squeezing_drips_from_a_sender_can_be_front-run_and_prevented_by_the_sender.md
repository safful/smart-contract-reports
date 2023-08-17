## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- M-01

# [Squeezing drips from a sender can be front-run and prevented by the sender](https://github.com/code-423n4/2023-01-drips-findings/issues/276) 

# Lines of code

https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Drips.sol#L411
https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Drips.sol#L461


# Vulnerability details

Squeezing drips from a sender requires providing the sequence of drips configurations (see NatSpec description in [L337-L338](https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Drips.sol#L337-L338)):

> /// It can start at an arbitrary past configuration, but must describe all the configurations
> /// which have been used since then including the current one, in the chronological order.

The provided history entries are hashed and verified against the sender's `dripsHistoryHash`.

However, the sender can prevent a receiver from squeezing drips by front-running the squeeze transaction and adding a new configuration. Adding a new configuration updates the current `dripsHistoryHash` and invalidates the `historyHash` provided by the receiver when squeezing. The receiver will then fail the drips history verification in [L461](https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Drips.sol#L461) and the squeeze will fail.

## Impact

A sender can prevent its drip receivers from squeezing by front-running the squeeze transaction and adding a new configuration.

## Proof of Concept

[Drips.sol#L411](https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Drips.sol#L411)

```solidity
392: function _squeezeDripsResult(
393:     uint256 userId,
394:     uint256 assetId,
395:     uint256 senderId,
396:     bytes32 historyHash,
397:     DripsHistory[] memory dripsHistory
398: )
399:     internal
400:     view
401:     returns (
402:         uint128 amt,
403:         uint256 squeezedNum,
404:         uint256[] memory squeezedRevIdxs,
405:         bytes32[] memory historyHashes,
406:         uint256 currCycleConfigs
407:     )
408: {
409:     {
410:         DripsState storage sender = _dripsStorage().states[assetId][senderId];
411:         historyHashes = _verifyDripsHistory(historyHash, dripsHistory, sender.dripsHistoryHash);
412:         // If the last update was not in the current cycle,
413:         // there's only the single latest history entry to squeeze in the current cycle.
414:         currCycleConfigs = 1;
415:         // slither-disable-next-line timestamp
416:         if (sender.updateTime >= _currCycleStart()) currCycleConfigs = sender.currCycleConfigs;
417:     }
...      // [...]
```

[Drips.sol#L461](https://github.com/code-423n4/2023-01-drips/blob/9fd776b50f4be23ca038b1d0426e63a69c7a511d/src/Drips.sol#L461)

```solidity
444: function _verifyDripsHistory(
445:     bytes32 historyHash,
446:     DripsHistory[] memory dripsHistory,
447:     bytes32 finalHistoryHash
448: ) private pure returns (bytes32[] memory historyHashes) {
449:     historyHashes = new bytes32[](dripsHistory.length);
450:     for (uint256 i = 0; i < dripsHistory.length; i++) {
451:         DripsHistory memory drips = dripsHistory[i];
452:         bytes32 dripsHash = drips.dripsHash;
453:         if (drips.receivers.length != 0) {
454:             require(dripsHash == 0, "Drips history entry with hash and receivers");
455:             dripsHash = _hashDrips(drips.receivers);
456:         }
457:         historyHashes[i] = historyHash;
458:         historyHash = _hashDripsHistory(historyHash, dripsHash, drips.updateTime, drips.maxEnd);
459:     }
460:     // slither-disable-next-line incorrect-equality,timestamp
461:     require(historyHash == finalHistoryHash, "Invalid drips history");
462: }
```

## Tools Used

Manual review

## Recommended mitigation steps

Consider allowing a receiver to squeeze drips from a sender up until the current timestamp.
