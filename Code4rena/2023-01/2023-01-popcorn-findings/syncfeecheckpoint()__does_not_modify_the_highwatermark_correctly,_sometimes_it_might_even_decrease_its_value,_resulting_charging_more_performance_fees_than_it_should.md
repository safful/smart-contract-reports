## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- edited-by-warden
- M-23

# [syncFeeCheckpoint()  does not modify the highWaterMark correctly, sometimes it might even decrease its value, resulting charging more performance fees than it should](https://github.com/code-423n4/2023-01-popcorn-findings/issues/365) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L496-L499


# Vulnerability details

## Impact
Detailed description of the impact of this finding.
``syncFeeCheckpoint()``  does not modify the ``highWaterMark`` correctly, sometimes it might even decrease its value, resulting charging more performance fees than it should.


## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.

The ``Vault.syncFeeCheckpoint()`` function does not modify the ``highWaterMark`` correctly, sometimes it might even decrease its value, resulting charging more performance fees than it should.  Instead of updating with a higher share values, it might actually decrease the value of ``highWaterMark``. As a result more performance fees might be charged since the ``highWaterMark`` was brought down again and again. 

[https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#
L496-L499](https://github.com/code-423n4/2023-01-popcorn/blob/d95fc31449c260901811196d617366d6352258cd/src/vault/Vault.sol#L496-L499)



```
 modifier syncFeeCheckpoint() {
        _;
        highWaterMark = convertToAssets(1e18);
    }
```

1) Suppose the current ``highWaterMark = 2 * e18`` and ``convertToAssets(1e18) = 1.5 * e18``. 

2) After ``deposit()`` is called, since the ``deposit()`` function has the ``synFeeCheckpoint`` modifier, the ``highWaterMark`` will be incorrectly reset to ``1.5 * e18``.

3) Suppose after some activities, ``convertToAssets(1e18) = 1.99 * e18``.  
 
4) ``TakeFees()`` is called, then the performance fee will be charged, since it wrongly decides ``convertToAssets(1e18) > highWaterMark`` with the wrong ``highWaterMark = 1.5 * e18``. The correct ``highWaterMark`` should be ``2 * e18``

```javascript
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
5) As a result, the performance fee is charged when it is not supposed to do so. Investors might not be happy with this.

## Tools Used
Remix

## Recommended Mitigation Steps
Revise the ``syncFeeCheckpoint()`` as follows:


```
 modifier syncFeeCheckpoint() {
        _;
     
         uint256 shareValue = convertToAssets(1e18);

        if (shareValue > highWaterMark) highWaterMark = shareValue;
    }
```