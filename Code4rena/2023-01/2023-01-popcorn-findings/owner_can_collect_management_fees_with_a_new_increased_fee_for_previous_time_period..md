## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-19

# [Owner can collect management fees with a new increased fee for previous time period.](https://github.com/code-423n4/2023-01-popcorn-findings/issues/466) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L480
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L488
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L540
https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/Vault.sol#L429


# Vulnerability details

## Impact
The Owner of the contract can change the fee, and after the rage quit period it can cash those fees for the period of time where different fees were active.
This allows the Owner to collect as much management fee as it wants for an already passed time period, when it should only apply from the time it has been changed.

## Proof of Concept
This contract allows the Owner to change the fees. To do so the ``proposeFees()`` procedure is called. This procedure will update the proposed fees, which then can be applied by calling the ``changeFees()`` procedure after the rage quit period has passed (the rage quit period is a period of time that has to pass between proposing a fee and applying it).

To collect the fees, the Owner calls the procedure ``takeManagementAndPerformanceFees()``. This procedure contains the modifier ``takeFees()`` which collects the fees.
```
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

Inside the ``takeFees()`` modifier the code ``if (managementFee > 0) feesUpdatedAt = block.timestamp;`` updates the variable ``feesUpdatedAt`` only if the fees are greater than 0. This variable is used as a timestamp of when was the las time fees were collected.
``managementFee`` is calculated using the function ``accruedManagementFees()``.

```
function accruedManagementFee() public view returns (uint256) {
        uint256 managementFee = fees.management;
        return
            managementFee > 0
                ? managementFee.mulDiv(
                    totalAssets() * (block.timestamp - feesUpdatedAt),
                    SECONDS_PER_YEAR,
                    Math.Rounding.Down
                ) / 1e18
                : 0;
    }
```


Here the ``managementFee`` is calculated using the ``feesUpdatedAt`` variable. In the calculation, the further apart the ``feesUpdated`` is in comparison to the ``block.timestamp``, the greater the fee will be. 

This ``feesUpdatedAt`` variable is not updated when the fee is changed using ``changeFee()`` or when the fees are proposed using ``proposeFee()``.
This allows the owner to collect a fee for a period of time where the fee was different.



**Current behaviour:**
```
                       Change fee to 2
                              |
                              v            Collect fees with
  Fee = 0                   Fee = 2        managementFee = 2
    |                         |                    |
    v                         v                    v
  -----------------------------------------------------
    ^                         ^                    ^
    |                         |                    |
| Day 0                     Day 5                 Day 7 |
|                                                       |
+-------------------------------------------------------+
                     Period of time of fees
                     collected with
                     management fee = 2
```

**Ideal behaviour:**
```
                            Change fee to 2
                                  |
                                  v            Collect fees with
      Fee = 0                   Fee = 2        managementFee = 2
        |                         |                    |
        v                         v                    v
      -----------------------------------------------------
        ^                         ^                    ^
        |                         |                    |
    | Day 0                     Day 5                 Day 7 |
    |                             |                         |
    +-----------------------------+-------------------------+
       Period of time of fees        Period of time of fees
       collected with                collected with
       management fee = 0            management fee = 2



```

The diagrams show how the management fees are charged for a period of time where they were not active.
The ideal behaviour diagram shows how it should behave in order to apply the management fees fairly.


 
**Steps:**

1. Create vault with 0 fees
2. Wait x amount of days
3. Change fee to something bigger than the previous fee
4. Collect the fees (fees collected will be from creation of vault until now with the new fee)

With this 4 steps a Vault creator can charge a new management fee for period of time where a different fee was active.



## Tools Used
Visual Studio Code

## Recommended Mitigation Steps
The variable ``feesUpdatedAt`` should be updated even when the fees are 0.
Fees should be collected when a new fee is applied, therefore the time period where the former fee was current will be collected correctly and the ``feesUpdatedAt`` variable will be updated to the timestamp of when the new fee has been applied.