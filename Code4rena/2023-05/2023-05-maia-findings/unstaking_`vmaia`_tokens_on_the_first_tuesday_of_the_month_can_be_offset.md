## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-24

# [Unstaking `vMAIA` tokens on the first Tuesday of the month can be offset](https://github.com/code-423n4/2023-05-maia-findings/issues/469) 

# Lines of code

https://github.com/code-423n4/2023-05-maia/blob/main/src/maia/vMaia.sol#L97-L114


# Vulnerability details

## Impact

According to [project documentation](https://v2-docs.maiadao.io/protocols/overview/tokenomics/vMaia) and [natspec](https://github.com/code-423n4/2023-05-maia/blob/main/src/maia/vMaia.sol#L23-L24):
> Users can stake their MAIA tokens at any time, but can only withdraw their staked tokens on the first Tuesday of each month. 

> NOTE: Withdraw is only allowed once per month, during the 1st Tuesday (UTC+0) of the month.

The implementation that keeps the above invariant true is dependent on at least one user attempting to unstake their `vMAIA` on the first chronological Tuesday of the month. But if nobody unstakes on the first Tuesday, then, on the second Tuesday of the month, the conditions are met and users can unstake then. Again, if no one unstakes on the second Tuesday, then the next Tuesday after that will be valid. So on and so forth.

Not respecting declared withdraw/unstaking period and limitation is a sever protocol issue in itself. The case is also not that improbable to happen, if good enough incentives are present there will be odd Tuesdays where nobody will unstake, thus creating this loophole.

### Issue details

`vMAIA` is an `ERC4626` vault compliant contract ([`vMAIA`](https://github.com/code-423n4/2023-05-maia/blob/main/src/maia/vMaia.sol#L26) -> [`ERC4626PartnerManager`](https://github.com/code-423n4/2023-05-maia/blob/main/src/maia/tokens/ERC4626PartnerManager.sol#L22) -> [`ERC4626`](https://github.com/code-423n4/2023-05-maia/blob/main/src/erc-4626/ERC4626.sol)). `ERC4626::withdraw` has a [beforeWithdraw](https://github.com/code-423n4/2023-05-maia/blob/main/src/erc-4626/ERC4626.sol#L70) hook callback that is overwritten/implemented in [vMAIA::beforeWithdraw](https://github.com/code-423n4/2023-05-maia/blob/main/src/maia/vMaia.sol#L97-L114)

```Solidity
    /**
     * @notice Function that performs the necessary verifications before a user can withdraw from their vMaia position.
     *  Checks if we're inside the unstaked period, if so then the user is able to withdraw.
     * If we're not in the unstake period, then there will be checks to determine if this is the beginning of the month.
     */
    function beforeWithdraw(uint256, uint256) internal override {
        /// @dev Check if unstake period has not ended yet, continue if it is the case.
        if (unstakePeriodEnd >= block.timestamp) return;


        uint256 _currentMonth = DateTimeLib.getMonth(block.timestamp);
        if (_currentMonth == currentMonth) revert UnstakePeriodNotLive();


        (bool isTuesday, uint256 _unstakePeriodStart) = DateTimeLib.isTuesday(block.timestamp);
        if (!isTuesday) revert UnstakePeriodNotLive();


        currentMonth = _currentMonth;
        unstakePeriodEnd = _unstakePeriodStart + 1 days;
    }
```

By thoroughly analyzing the function we can see that:

- it first checks if unstake period has not ended. The unstake period is 24h since the start of Tuesday. On the first call for the contract this is 0, so execution continues
```Solidity
        /// @dev Check if unstake period has not ended yet, continue if it is the case.
        if (unstakePeriodEnd >= block.timestamp) return;
```

- it then gets the current month and compares it to the last saved "currentMonth". The `currentMonth` is set only after the Tuesday condition is meet. Doing it this way they assure that after a Tuesday was validated then no further unstakes can happen in the same month
```Solidity
        uint256 _currentMonth = DateTimeLib.getMonth(block.timestamp);
        if (_currentMonth == currentMonth) revert UnstakePeriodNotLive();
```

- the next operation is to determine if now is a Tuesday and also return the start of the current day (this is to be used in determining the unstake period). To note here is that the check is only it is a Tuesday, **not the first Tuesday of the month**, rather, up until now, the check is that  _this is the first Tuesday in a month that was noted by this execution_
```Solidity
        (bool isTuesday, uint256 _unstakePeriodStart) = DateTimeLib.isTuesday(block.timestamp);
        if (!isTuesday) revert UnstakePeriodNotLive();
```

- after checking that we are in _the first **marked** Tuesday of this month_ the current month is noted (saved to `currentMonth`) and the unstake period is defined as the entire day (24h since the start of Tuesday)
```Solidity
        currentMonth = _currentMonth;
        unstakePeriodEnd = _unstakePeriodStart + 1 days;
```

To conclude the flow: the withdraw limitation is actually: _in a given month, on the first Tuesday where users attempt to withdraw and only on that Tuesday will withdrawals be allowed. It can be the last Tuesday of the month or the first Tuesday of the month_.

## Proof of Concept

Add the following coded POC to `test\maia\vMaiaTest.t.sol` and run it with `forge test --match-test testWithdrawMaiaWorksOnAnyThursday -vvv`
```Solidity

    import {DateTimeLib as MaiaDateTimeLib} from "@maia/Libraries/DateTimeLib.sol"; // add this next to the other imports

    function testWithdrawMaiaWorksOnAnyThursday() public {
        testDepositMaia();
        uint256 amount = 100 ether;

        // we now are in the first Tuesday of the month (ignore the name, getFirstDayOfNextMonthUnix gets the first Tuesday of the month)
        hevm.warp(getFirstDayOfNextMonthUnix());

        // sanity check that we are actually in a Tuesday
        (bool isTuesday_, ) = MaiaDateTimeLib.isTuesday(block.timestamp);
        assertTrue(isTuesday_);

        // no withdraw is done, and then the next Tuesday comes
        hevm.warp(block.timestamp + 7 days);

        // sanity check that we are actually in a Tuesday, again
        (isTuesday_, ) = MaiaDateTimeLib.isTuesday(block.timestamp);
        assertTrue(isTuesday_);

        // withdraw succeeds even if we are NOT in the first Tuesday of the month, but in the second one
        vmaia.withdraw(amount, address(this), address(this));

        assertEq(maia.balanceOf(address(vmaia)), 0);
        assertEq(vmaia.balanceOf(address(this)), 0);
    }
```

## Tools Used

Manual analysis and ChatGPT for the `isFirstTuesdayOfMonth` function optimizations.

## Recommended Mitigation Steps

Modify the `isTuesday` function into a `isFirstTuesdayOfMonth` function, a function that checks that the given timestamp is in the first Tuesday of it's containing month.

Example implementation:

```Solidity
    /// @dev Returns if the provided timestamp is in the first Tuesday of it's corresponding month (result) and (startOfDay);
    ///      startOfDay will always by the timestamp of the first Tuesday found searching from the given timestamp,     
    ///      regardless if it's the first of the month or not, so always check result if using it
    function isFirstTuesdayOfMonth(uint256 timestamp) internal pure returns (bool result, uint256 startOfDay) {
        uint256 month = getMonth(timestamp);
        uint256 firstDayOfMonth = timestamp - ((timestamp % 86400) + 1) * 86400;
        
        uint256 dayIndex = ((firstDayOfMonth / 86400 + 3) % 7) + 1; // Monday: 1, Tuesday: 2, ....., Sunday: 7.
        uint256 daysToAddToReachNextTuesday = (9 - dayIndex) % 7;

        startOfDay = firstDayOfMonth + daysToAddToReachNextTuesday * 86400;
        result = (startOfDay <= timestamp && timestamp < startOfDay + 86400) && month == getMonth(startOfDay);
    }

```


## Assessed type

Timing