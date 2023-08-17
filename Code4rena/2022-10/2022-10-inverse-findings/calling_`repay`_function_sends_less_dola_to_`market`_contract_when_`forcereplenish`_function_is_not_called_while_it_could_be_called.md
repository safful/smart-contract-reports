## Tags

- bug
- 2 (Med Risk)
- primary issue
- sponsor disputed
- selected for report
- M-16

# [Calling `repay` function sends less DOLA to `Market` contract when `forceReplenish` function is not called while it could be called](https://github.com/code-423n4/2022-10-inverse-findings/issues/583) 

# Lines of code

https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L559-L572
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L531-L539
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L472-L474
https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L460-L466


# Vulnerability details

## Impact
When a user incurs a DBR deficit, a replenisher can call the `forceReplenish` function to force the user to replenish DBR. However, there is no guarantee that the `forceReplenish` function will always be called. When the `forceReplenish` function is not called, such as because that the replenisher does not notice the user's DBR deficit promptly, the user can just call the `repay` function to repay the origianl debt and the `withdraw` function to receive all of the deposited collateral even when the user has a DBR deficit already. Yet, in the same situation, if the `forceReplenish` function has been called, more debt should be added for the user, and the user needs to repay more in order to get back all of the deposited collateral. Hence, when the `forceReplenish` function is not called while it could be called, the `Market` contract would receive less DOLA if the user decides to repay the debt and withdraw the collateral both in full.

https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L559-L572
```solidity
    function forceReplenish(address user, uint amount) public {
        uint deficit = dbr.deficitOf(user);
        require(deficit > 0, "No DBR deficit");
        require(deficit >= amount, "Amount > deficit");
        uint replenishmentCost = amount * dbr.replenishmentPriceBps() / 10000;
        uint replenisherReward = replenishmentCost * replenishmentIncentiveBps / 10000;
        debts[user] += replenishmentCost;
        uint collateralValue = getCollateralValueInternal(user);
        require(collateralValue >= debts[user], "Exceeded collateral value");
        totalDebt += replenishmentCost;
        dbr.onForceReplenish(user, amount);
        dola.transfer(msg.sender, replenisherReward);
        emit ForceReplenish(user, msg.sender, amount, replenishmentCost, replenisherReward);
    }
```

https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L531-L539
```solidity
    function repay(address user, uint amount) public {
        uint debt = debts[user];
        require(debt >= amount, "Insufficient debt");
        debts[user] -= amount;
        totalDebt -= amount;
        dbr.onRepay(user, amount);
        dola.transferFrom(msg.sender, address(this), amount);
        emit Repay(user, msg.sender, amount);
    }
```

https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L472-L474
```solidity
    function withdraw(uint amount) public {
        withdrawInternal(msg.sender, msg.sender, amount);
    }
```

https://github.com/code-423n4/2022-10-inverse/blob/main/src/Market.sol#L460-L466
```solidity
    function withdrawInternal(address from, address to, uint amount) internal {
        uint limit = getWithdrawalLimitInternal(from);
        require(limit >= amount, "Insufficient withdrawal limit");
        IEscrow escrow = getEscrow(from);
        escrow.pay(to, amount);
        emit Withdraw(from, to, amount);
    }
```

## Proof of Concept
Please add the following test in `src\test\Market.t.sol`. This test will pass to demonstrate the described scenario.

```solidity
    function testRepayAndWithdrawInFullWhenIncurringDBRDeficitIfNotBeingForcedToReplenish() public {
        gibWeth(user, wethTestAmount);
        gibDBR(user, wethTestAmount);

        vm.startPrank(user);

        // user deposits wethTestAmount WETH and borrows wethTestAmount DOLA
        deposit(wethTestAmount);
        market.borrow(wethTestAmount);

        assertEq(DOLA.balanceOf(user), wethTestAmount);
        assertEq(WETH.balanceOf(user), 0);

        vm.warp(block.timestamp + 60 weeks);

        // after some time, user incurs DBR deficit
        assertGt(dbr.deficitOf(user), 0);

        // yet, since no one notices that user has a DBR deficit and forces user to replenish DBR,
        //   user is able to repay wethTestAmount DOLA that was borrowed previously and withdraw wethTestAmount WETH that was deposited previously
        market.repay(user, wethTestAmount);
        market.withdraw(wethTestAmount);

        vm.stopPrank();

        // as a result, user is able to get back all of the deposited WETH, which should not be possible if user has been forced to replenish DBR
        assertEq(DOLA.balanceOf(user), 0);
        assertEq(WETH.balanceOf(user), wethTestAmount);
    }
```

## Tools Used
VSCode

## Recommended Mitigation Steps
When calling the `repay` function, the user's DBR deficit can also be checked. If the user has a DBR deficit, an amount, which is similar to `replenishmentCost` that is calculated in the `forceReplenish` function, can be calculated; it can then be used to adjust the `repay` function's `amount` input for updating the states regarding the user's and total debts in the relevant contracts.