## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- primary issue
- M-05

# [setPendingRedemptionBalance() may cause the user's cash token to be lost](https://github.com/code-423n4/2023-01-ondo-findings/issues/82) 

# Lines of code

https://github.com/code-423n4/2023-01-ondo/blob/f3426e5b6b4561e09460b2e6471eb694efdd6c70/contracts/cash/CashManager.sol#L851-L876


# Vulnerability details

## Impact
setPendingRedemptionBalance()  are not check Old balances, resulting in the possibility of overwriting the new balance added by the user

## Proof of Concept
in setPendingRedemptionBalance()
MANAGER_ADMIN can adjust the amount of the cash token of user to be burned  in some cases: addressToBurnAmt[user]
Three main parameters are passed in.
```solidity
    address user,
    uint256 epoch,
    uint256 balance
```

Before modification will check epoch can not be greater than the currentEpoch, is can modify the currentEpoch user balance.
This has a problem:
The user is able to increase the addressToBurnAmt[user]  of currentEpoch by ```requestRedemption()```
This leaves open the possibility that the user may have unknowingly executed requestRedemption() before settingPendingRedemptionBalance(), causing the increased balance to be overwritten
For example:
currentEpoch = 1
Balance of alice: addressToBurnAmt[alice] = 50

1. The administrator finds something wrong, there is 10 less, so he wants to increase it by 10, so he calls setPendingRedemptionBalance (balance=60)
2. alice does not know the above operation and wants to increase the redemption by 100, so it executes requestRedemption(100), which is executed earlier than setPendingRedemptionBalance() because the gas price is set higher
3. The result is that the final balance of alice becomes only 60. change process: 50 => 150 => 60

The result is missing 100

Suggest adding oldBalance, not equal will revert

## Tools Used

## Recommended Mitigation Steps

adding oldBalance, not equal will revert

```solidity
  function setPendingRedemptionBalance(
    address user,
    uint256 epoch,
+   uint256 oldBalance    
    uint256 balance
  ) external updateEpoch onlyRole(MANAGER_ADMIN) {
    if (epoch > currentEpoch) {
      revert CannotServiceFutureEpoch();
    }
+   require(oldBalance == redemptionInfoPerEpoch[epoch].addressToBurnAmt[user],"bad old balance");

```