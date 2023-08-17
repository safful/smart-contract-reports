## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- primary issue
- satisfactory
- selected for report
- edited-by-warden
- M-15

# [Borrow rate calculation can cause VToken.accrueInterest() to revert, DoSing all major functionality](https://github.com/code-423n4/2023-05-venus-findings/issues/10) 

# Lines of code

https://github.com/code-423n4/2023-05-venus/blob/main/contracts/VToken.sol#L695-L696


# Vulnerability details

## Impact
Borrow rates are calculated dynamically and VToken.accrueInterest() [reverts](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/VToken.sol#L695-L696) if the calculated rate is greater than a hard-coded maximum. As accrueInterest() is called by most VToken functions, this state causes a major DoS.

## Proof of Concept
VToken [hard-codes](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/VTokenInterfaces.sol#L53) the maximum borrow rate and accrueInterest() [reverts](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/VToken.sol#L695-L696) if the dynamically calculated rate is greater than the hard-coded value.

The actual calculation is dynamic [[1](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/BaseJumpRateModelV2.sol#L172-L190), [2](https://github.com/code-423n4/2023-05-venus/blob/main/contracts/WhitePaperInterestRateModel.sol#L50-L57)] and takes no notice of the hard-coded cap, so it is very possible that this state will manifest, causing a major DoS due to most VToken functions calling accrueInterest() and accrueInterest() reverting.

## Tools Used
Manual review

## Recommended Mitigation Steps
Change VToken.accrueInterest() to not revert in this case but simply to set borrowRateMantissa = borrowRateMaxMantissa if the dynamically calculated value would be greater than the hard-coded max. This would:

1) allow execution to continue operating with the system-allowed maximum borrow rate, allowing all functionality that depends upon accrueInterest() to continue as normal,

2) allow borrowRateMantissa to be naturally set to the dynamically calculated rate as soon as that rate becomes less than the hard-coded max. 






## Assessed type

DoS