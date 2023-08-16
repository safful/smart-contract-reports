## Tags

- bug
- 1 (Low Risk)
- resolved
- sponsor confirmed

# [On updating the Incentive fee greater than UFixedLib18.ONE, new Programs can not be created](https://github.com/code-423n4/2021-12-perennial-findings/issues/72) 

# Handle

hubble


# Vulnerability details

## Impact
Incentivizer.updateFee expects a value between UFixed18Lib.ZERO and UFixed18Lib.ONE. When the incentive fee is updated outside of this range, the program creation fails and the product owners would not be able to add new Programs to the Product.

## Proof of Concept
Step 1. Update incentive fee
File: /protocol/contracts/incentivizer/Incentivizer.sol
Line 368     function updateFee(UFixed18 newFee) onlyOwner external {
Line 369             fee = newFee;
...

Step 2. Create new Program
File: /protocol/contracts/incentivizer/Incentivizer.sol
Line 59    function create(ProgramInfo calldata info)

File: /protocol/contracts/incentivizer/types/ProgramInfo.sol
Line 55         Position memory amountAfterFee = info.amount.mul(UFixed18Lib.ONE.sub(fee));
...

Note: Fails at line 55, whenever fee is greater than UFixedLive.ONE

## Tools Used
Manual code review

## Recommended Mitigation Steps
Implement Range check 

File: /protocol/contracts/incentivizer/Incentivizer.sol
Line 368     function updateFee(UFixed18 newFee) onlyOwner external {
Line 369          if(newFee.gte(UFixed18Lib.ONE)) revert NewError("newFee should be less than UFixed18Lib.ONE");
Line 370          fee = newFee;

Note: Range check at line 369

