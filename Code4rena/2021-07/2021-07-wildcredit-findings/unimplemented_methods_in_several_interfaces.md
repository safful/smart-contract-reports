## Tags

- bug
- 0 (Non-critical)
- disagree with severity
- sponsor confirmed

# [Unimplemented methods in several interfaces](https://github.com/code-423n4/2021-07-wildcredit-findings/issues/140) 

# Handle

shw


# Vulnerability details

## Impact

Some methods declared in the interfaces are not implemented. Specifically, the `withdrawRepay` method of `ILendingPair` and the `liqFeePool` method of `Controller`.

## Proof of Concept

Referenced code:
[IController.sol#L14](https://github.com/code-423n4/2021-07-wildcredit/blob/main/contracts/interfaces/IController.sol#L14)
[ILendingPair.sol#L22](https://github.com/code-423n4/2021-07-wildcredit/blob/main/contracts/interfaces/ILendingPair.sol#L22)

## Recommended Mitigation Steps

Remove the unimplemented methods.

