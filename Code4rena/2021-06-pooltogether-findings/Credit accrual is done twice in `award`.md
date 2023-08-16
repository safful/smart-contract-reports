## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Credit accrual is done twice in `award`](https://github.com/code-423n4/2021-06-pooltogether-findings/issues/96) 

# Handle

cmichel


# Vulnerability details

The credit is accrued twice in `award`.
The first accrual happens implicitly when calling `_mint` through the `ControlledToken(controlledToken).controllerMint` call which then performs the `PrizePool.beforeTokenTransfer` hook which accrues credit.
Then the explicit accrual is done again. It should be enough to only add the `extraCredit` without doing another accrual (calling `_updateCreditBalance(..., newBalance= _applyCreditLimit(controlledToken, controlledTokenBalance, uint256(creditBalance.balance).add(credit).add(extra)))` instead).

