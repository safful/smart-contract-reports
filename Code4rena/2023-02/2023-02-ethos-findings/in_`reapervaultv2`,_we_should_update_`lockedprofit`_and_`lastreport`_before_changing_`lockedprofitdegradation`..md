## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- M-04

# [In `ReaperVaultV2`, we should update `lockedProfit` and `lastReport` before changing `lockedProfitDegradation`.](https://github.com/code-423n4/2023-02-ethos-findings/issues/728) 

# Lines of code

https://github.com/code-423n4/2023-02-ethos/blob/73687f32b934c9d697b97745356cdf8a1f264955/Ethos-Vault/contracts/ReaperVaultV2.sol#L617


# Vulnerability details

## Impact
The locked profit degradation for the past will be changed with the new `lockedProfitDegradation`.

As a result, malicious users can steal others' rewards by frontrunning.

## Proof of Concept
`setLockedProfitDegradation()` is used to change `lockedProfitDegradation` by admin.

```solidity
    function setLockedProfitDegradation(uint256 degradation) external {
        _atLeastRole(ADMIN);
        require(degradation <= DEGRADATION_COEFFICIENT, "Degradation cannot be more than 100%");
        lockedProfitDegradation = degradation;
        emit LockedProfitDegradationUpdated(degradation);
    }
```

And `lockedProfitDegradation` is used to calculate the locked profit.

```solidity
    function _calculateLockedProfit() internal view returns (uint256) {
        uint256 lockedFundsRatio = (block.timestamp - lastReport) * lockedProfitDegradation;
        if (lockedFundsRatio < DEGRADATION_COEFFICIENT) {
            return lockedProfit - ((lockedFundsRatio * lockedProfit) / DEGRADATION_COEFFICIENT);
        }

        return 0;
    }
```

But it doesn't update `lockedProfit` and `lastReport` before changing `lockedProfitDegradation` so the below scenario would be possible.

1. Let's assume `lockedProfit = 200, lastReport = block.timestamp` after calling `report()`, `lockedProfitDegradation` are `6 hours in blocks`.
2. 3 hours later, 100 tokens of `lockedProfit` are released and added to the free funds. We can assume `report()` wasn't called for 3 hours.
3. At that time, `lockedProfitDegradation` is changed to `4 hours in blocks` and it means `200 * 3 / 4 = 150` tokens are released. As a result, free funds are increased by 50 inside the same block.
4. So a malicious user(it should be a pool) can front run `deposit()` with huge amounts before `lockedProfitDegradation` is changed and charge most of the new rewards(=50).

Similarly, already unlocked funds will be treated as locked again if `lockedProfitDegradation` is decreased.

Even if there is no front run as all depositors are pools, it's not fair to change locked/unlocked amounts that are confirmed already.

## Tools Used
Manual Review

## Recommended Mitigation Steps
We should modify `setLockedProfitDegradation()` like below.

```solidity
    function setLockedProfitDegradation(uint256 degradation) external {
        _atLeastRole(ADMIN);
        require(degradation <= DEGRADATION_COEFFICIENT, "Degradation cannot be more than 100%");

        // update lockedProfit and lastReport
        lockedProfit = _calculateLockedProfit();
        lastReport = block.timestamp;

        lockedProfitDegradation = degradation;
        emit LockedProfitDegradationUpdated(degradation);
    }
```