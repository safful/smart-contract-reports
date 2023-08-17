## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-12

# [Fix utilization rate computation](https://github.com/code-423n4/2023-05-venus-findings/issues/122) 

# Lines of code

https://github.com/code-423n4/2023-05-venus/blob/8be784ed9752b80e6f1b8b781e2e6251748d0d7e/contracts/BaseJumpRateModelV2.sol#L131


# Vulnerability details

## Impact
The BaseJumpRateModelV2.sol#L131.utilizationRate() function can return value above 1 and not between [0, BASE].

## Proof of Concept
In The BaseJumpRateModelV2.sol#L131.utilizationRate() function, cash and borrows and reserves values gets used to calculate utilization rate between between [0, 1e18]. reserves is currently unused but it will be used in the future. 

     */
    function utilizationRate(
        uint256 cash,
        uint256 borrows,
        uint256 reserves
    ) public pure returns (uint256) {
        // Utilization rate is 0 when there are no borrows
        if (borrows == 0) {
            return 0;
        }

        return (borrows * BASE) / (cash + borrows - reserves);
    }

If Borrow value is 0, then function will return 0. but in this function the scenario where the value of reserves exceeds cash is not handled. the system does not guarantee that reserves never exceeds cash. the reserves grow automatically over time, so it might be difficult to avoid this entirely. 

If reserves > cash (and borrows + cash - reserves > 0), the formula for utilizationRate above gives a utilization rate above 1.

## Tools Used
Manually

## Recommended Mitigation Steps
Make the utilization rate computation return 1 if reserves > cash.


## Assessed type

Math