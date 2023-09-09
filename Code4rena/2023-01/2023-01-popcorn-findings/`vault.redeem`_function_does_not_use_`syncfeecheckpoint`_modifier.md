## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-10

# [`Vault.redeem` function does not use `syncFeeCheckpoint` modifier](https://github.com/code-423n4/2023-01-popcorn-findings/issues/557) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L253-L278
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L211-L240
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L496-L499
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L473-L477
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L480-L494
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L447-L460


# Vulnerability details

## Impact
The following `Vault.redeem` function does not use the `syncFeeCheckpoint` modifier, which is unlike the `Vault.withdraw` function below. Because of this, after calling the `Vault.redeem` function, `highWaterMark` is not sync'ed. In this case, calling functions like `Vault.takeManagementAndPerformanceFees` after the `Vault.redeem` function is called and before the `syncFeeCheckpoint` modifier is triggered will eventually use a stale `highWaterMark` to call the `Vault.accruedPerformanceFee` function. This will cause the performance fee to be calculated inaccurately in which the `feeRecipient` can receive more performance fee than it should receive or receive no performance fee when it should.

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L253-L278
```solidity
    function redeem(
        uint256 shares,
        address receiver,
        address owner
    ) public nonReentrant returns (uint256 assets) {
        ...
    }
```

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L211-L240
```solidity
    function withdraw(
        uint256 assets,
        address receiver,
        address owner
    ) public nonReentrant syncFeeCheckpoint returns (uint256 shares) {
        ...
    }
```

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L496-L499
```solidity
    modifier syncFeeCheckpoint() {
        _;
        highWaterMark = convertToAssets(1e18);
    }
```

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L473-L477
```solidity
    function takeManagementAndPerformanceFees()
        external
        nonReentrant
        takeFees
    {}
```

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L480-L494
```solidity
    modifier takeFees() {
        uint256 managementFee = accruedManagementFee();
        uint256 totalFee = managementFee + accruedPerformanceFee();
        uint256 currentAssets = totalAssets();
        uint256 shareValue = convertToAssets(1e18);

        if (shareValue > highWaterMark) highWaterMark = shareValue;

        if (managementFee > 0) feesUpdatedAt = block.timestamp;

        if (totalFee > 0 && currentAssets > 0)
            _mint(feeRecipient, convertToShares(totalFee));

        _;
    }
```

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L447-L460
```solidity
    function accruedPerformanceFee() public view returns (uint256) {
        uint256 highWaterMark_ = highWaterMark;
        uint256 shareValue = convertToAssets(1e18);
        uint256 performanceFee = fees.performance;

        return
            performanceFee > 0 && shareValue > highWaterMark
                ? performanceFee.mulDiv(
                    (shareValue - highWaterMark) * totalSupply(),
                    1e36,
                    Math.Rounding.Down
                )
                : 0;
    }
```


## Proof of Concept
The following steps can occur for the described scenario.
1. A user calls the `Vault.redeem` function, which does not sync `highWaterMark`.
2. The vault owner calls the `Vault.takeManagementAndPerformanceFees` function, which eventually calls the `accruedPerformanceFee` function.
3. When calling the `Vault.accruedPerformanceFee` function, because `convertToAssets(1e18)` is less than the stale `highWaterMark`, no performance fee is accrued. If calling the `Vault.redeem` function can sync `highWaterMark`, some performance fee would be accrued through using such updated `highWaterMark` but that is not the case here.
4. `feeRecipient` receives no performance fee when it should.

## Tools Used
VSCode

## Recommended Mitigation Steps
The `Vault.redeem` function can be updated to use the `syncFeeCheckpoint` modifier.