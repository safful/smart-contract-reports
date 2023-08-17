## Tags

- bug
- 2 (Med Risk)
- judge review requested
- satisfactory
- selected for report
- sponsor disputed
- M-09

# [Withdrawals will stuck](https://github.com/code-423n4/2023-01-reserve-findings/issues/325) 

# Lines of code

https://github.com/reserve-protocol/protocol/blob/df7ecadc2bae74244ace5e8b39e94bc992903158/contracts/p1/StRSR.sol#L397


# Vulnerability details

## Impact
If a new era gets started for stakeRSR and draftRSR still point to old era then user will be at risk of losing their future holdings

## Proof of Concept
1. seizeRSR is called with amount 150 where stakeRSR was 50 and draftRSR was 80. The era was 1 currently for both stake and draft
```
function seizeRSR(uint256 rsrAmount) external notPausedOrFrozen {
...
stakeRSR -= stakeRSRToTake;
..
if (stakeRSR == 0 || stakeRate > MAX_STAKE_RATE) {
            seizedRSR += stakeRSR;
            beginEra();
        }
...
draftRSR -= draftRSRToTake;
if (draftRSR > 0) {
            // Downcast is safe: totalDrafts is 1e38 at most so expression maximum value is 1e56
            draftRate = uint192((FIX_ONE_256 * totalDrafts + (draftRSR - 1)) / draftRSR);
        }

...
```

2. stakeRSR portion comes to be 50 which means remaining stakeRSR will be 0 (50-50). This means a new staking era will get started

```
if (stakeRSR == 0 || stakeRate > MAX_STAKE_RATE) {
            seizedRSR += stakeRSR;
            beginEra();
        }
```

3. This causes staking era to become 2

```
function beginEra() internal virtual {
        stakeRSR = 0;
        totalStakes = 0;
        stakeRate = FIX_ONE;
        era++;

        emit AllBalancesReset(era);
    }
```

4. Now draftRSR is still > 0 so only draftRate gets updated. The draft Era still remains 1

5. User stakes and unstakes in this new era. Staking is done in era 2

6. Unstaking calls the pushDraft which creates User draft on draftEra which is still 1

```
function pushDraft(address account, uint256 rsrAmount)
        internal
        returns (uint256 index, uint64 availableAt)
    {
...
CumulativeDraft[] storage queue = draftQueues[draftEra][account];
       ...

        queue.push(CumulativeDraft(uint176(oldDrafts + draftAmount), availableAt));
...
}
```

7. Lets say due to unfortunate condition again seizeRSR need to be called. This time draftRate > MAX_DRAFT_RATE which means draft era increases and becomes 2

```
if (draftRSR == 0 || draftRate > MAX_DRAFT_RATE) {
            seizedRSR += draftRSR;
            beginDraftEra();
        }
```

8. This becomes a problem since all unstaking done till now in era 2 were pointing in draft era 1. Once draft era gets updated to 2, all those unstaking are lost

## Recommended Mitigation Steps
Era should be same for staking and draft. So if User is unstaking at era 1 then withdrawal draft should always be era 1 and not some previous era