## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-07

# [Improper Approval Mechanism of Clearing House](https://github.com/code-423n4/2023-01-astaria-findings/issues/472) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/ClearingHouse.sol#L148-L151
https://github.com/code-423n4/2023-01-astaria/blob/main/src/ClearingHouse.sol#L160-L165
https://github.com/code-423n4/2023-01-astaria/blob/main/src/LienToken.sol#L637-L641
https://github.com/code-423n4/2023-01-astaria/blob/main/src/CollateralToken.sol#L566-L577


# Vulnerability details

## Impact

The `ClearingHouse` implementation performs a `@solmate`-based `safeApprove` instruction ([1]) with the remaining `balanceOf(address(this))` but contains code handling any remainder of funds that may remain in the contract [2]. Investigation of the `payDebtViaClearingHouse` function will indicate that the function may not consume the maximum approval that was set to the `TRANSFER_PROXY` [3]. 

As a result, any consequent invocation of `_execute` via `safeTransferFrom` from OpenSea with a `paymentToken` (i.e. `identifier`) such as USDT would fail. Given that the `ClearingHouse` of a particular `collateralId` is created only once, this vulnerability will impact consequent listings and cause them to fatally fail for a token that has been used in the past and is part of the non-compliant EIP-20 tokens, with USDT being the prime and most popular example.

The code in USDT that causes this complication is as follows:

```sol
require(!((_value != 0) && (allowed[msg.sender][_spender] != 0)));
```

## Proof of Concept

The vulnerability is clearly defined above, however, for testing purposes, the `ClearingHouse::safeTransferFrom` function of a particular clearing house instance can be invoked twice with the same arguments. The second invocation will fail provided that the first invocation provided a token balance that exceeds the number of funds necessary for the debt payment which should be denoted in USDT.

On an important note, if the code used `safeApprove` from `@openzeppelin` this issue would affect any token, however, it is limited to non-standard tokens due to the usage of the `@solmate` implementation of `safeApprove`.

## Tools Used

Manual review of the codebase. Historically, findings pertaining to incorrect approval mechanisms that do not support USDT have been marked as "medium" in severity in the past in the following cases:

- Rubicon: https://code4rena.com/reports/2022-05-rubicon/#m-08-usdt-is-not-supported-because-of-approval-mechanism
- Duality Focus: https://code4rena.com/reports/2022-04-dualityfocus/#m-03-not-calling-approve0-before-setting-a-new-approval-causes-the-call-to-revert-when-used-with-tether-usdt

## Recommended Mitigation Steps

We advise approvals to be properly handled by evaluating whether a non-zero approval already exists and in such an instance nullifying it before re-setting it to a non-zero value. Example below:

```sol
// Optimizing lookup
address transferProxy = address(ASTARIA_ROUTER.TRANSFER_PROXY());

// If existing approval is non-zero -> set it to zero
if (ERC20(paymentToken).allowance(address(this), transferProxy) != 0) {
    ERC20(paymentToken).safeApprove(transferProxy, 0);
}

// Set non-zero approval
ERC20(paymentToken).safeApprove(transferProxy, payment - liquidatorPayment);
```