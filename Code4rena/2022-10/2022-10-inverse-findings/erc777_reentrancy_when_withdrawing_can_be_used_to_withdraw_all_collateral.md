## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- satisfactory
- sponsor acknowledged
- selected for report
- M-04

# [ERC777 reentrancy when withdrawing can be used to withdraw all collateral](https://github.com/code-423n4/2022-10-inverse-findings/issues/206) 

# Lines of code

https://github.com/code-423n4/2022-10-inverse/blob/cc281e5800d5860c816138980f08b84225e430fe/src/Market.sol#L464


# Vulnerability details

## Impact
Markets can be deployed with arbitrary tokens for the collateral, including ERC777 tokens (that are downwards-compatible with ERC20). However, when the system is used with those tokens, an attacker can drain his escrow contract completely while still having a loan. This happens because with ERC777 tokens, there is a `tokensToSend` hook that is executed before the actual transfer (and the balance updates) happen. Therefore, `escrow.balance()` (which retrieves the token balance) will still report the old balance when an attacker reenters from this hook.

## Proof Of Concept
We assume that `collateral` is an ERC777 token and that the `collateralFactorBps` is 5,000 (50%). The user has deposited 10,000 USD (worth of collateral) and taken out a loan worth 2,500 USD. He is therefore allowed to withdraw 5,000 USD (worth of collateral). However, he can usse the ERC777 reentrancy to take out all 10,000 USD (worth of collateral) and still keep the loaned 2,500 USD:
1.) The user calls `withdraw(amount)` to withdraw his 5,000 USD (worth of collateral).
2.) In `withdrawInternal`, the limit check succeeds (the user is allowed to withdraw 5,000 USD) and `escrow.pay(to, amount)` is called. This will initiate a transfer to the provided address (no matter which escrow is used, but we assume `SimpleERC20Escrow` for this example).
3.) Because the collateral is an ERC777 token, the `tokensToSend` hook is executed before the actual transfer (and before any balance updates are made). The user can exploit this by calling `withdraw(amount)` again within the hook.
4.) `withdrawInternal` will call `getWithdrawalLimitInternal`, which calls `escrow.balance()`. This receives the collateral balance of the escrow, which is not yet updated. Because of that, the balance is still 10,000 USD (worth of collateral) and the calculated withdraw limit is therefore still 5,000 USD.
5.) Both transfers (the reentered one and the original one) succeed and the user has received all of his collateral (10,000 USD), while still having the 2,500 USD loan.

## Recommended Mitigation Steps
Mark these functions as `nonReentrant`.