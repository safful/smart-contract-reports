## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- selected for report
- sponsor confirmed
- M-02

# [`pause/unpause` functionnalities not implemented in many pausable contracts](https://github.com/code-423n4/2023-06-stader-findings/issues/383) 

# Lines of code

https://github.com/code-423n4/2023-06-stader/blob/main/contracts/SocializingPool.sol#L21
https://github.com/code-423n4/2023-06-stader/blob/main/contracts/Auction.sol#L14
https://github.com/code-423n4/2023-06-stader/blob/main/contracts/StaderOracle.sol#L17
https://github.com/code-423n4/2023-06-stader/blob/main/contracts/OperatorRewardsCollector.sol#L16


# Vulnerability details

## Impact

The following contracts : `SocializingPool`, `StaderOracle`, `OperatorRewardsCollector` and `Auction` are supposed to be pausable (as they all inherit from `PausableUpgradeable`) but they don't implement the external `pause/unpause` functionalities which means it will never be possible to pause them.

## Proof of Concept

All the following contracts `SocializingPool`, `StaderOracle`, `OperatorRewardsCollector` and `Auction` inherit from the openzeppelin `PausableUpgradeable` extension which means that they contain internal functions `_pause` and `_unpause`.

Because those function are internal, the contract must implement two other public/external `pause` and `unpause` functions to allow the manager to pause and unpause the contracts when necessary, but none of the aforementioned contracts implement those functions which means that even if those contracts are supposed to be pausable (have the `pause/unpause` functionalities) none of them can be paused.

## Tools Used

Manual review

## Recommended Mitigation Steps

Add public/external `pause` and `unpause` functions in the aforementioned contracts to allow them to be pausable, this can be done just as in the `UserWithdrawalManager` contract for example :

```solidity
/**
 * @dev Triggers stopped state.
 * Contract must not be paused
 */
function pause() external {
    UtilLib.onlyManagerRole(msg.sender, staderConfig);
    _pause();
}

/**
 * @dev Returns to normal state.
 * Contract must be paused
 */
function unpause() external onlyRole(DEFAULT_ADMIN_ROLE) {
    _unpause();
}
```


## Assessed type

Other