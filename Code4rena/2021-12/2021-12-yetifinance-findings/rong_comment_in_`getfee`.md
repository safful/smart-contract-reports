## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- sponsor confirmed

# [rong comment in `getFee`](https://github.com/code-423n4/2021-12-yetifinance-findings/issues/203) 

# Handle

cmichel


# Vulnerability details

The `ThreePieceWiseLinearPriceCurve.getFee` comment states that the total + the input must be less than the cap:

> If dollarCap == 0, then it is not capped. Otherwise, **then the total + the total input** must be less than the cap.

The code only checks if the input is less than the cap:

```solidity
// @param _collateralVCInput is how much collateral is being input by the user into the system
if (dollarCap != 0) {
    require(_collateralVCInput <= dollarCap, "Collateral input exceeds cap");
}
```

## Recommended Mitigation Steps
Clarify the desired behavior and reconcile the code with the comments.


