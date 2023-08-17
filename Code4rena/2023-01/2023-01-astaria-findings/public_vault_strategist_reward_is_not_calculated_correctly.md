## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-08

# [Public vault strategist reward is not calculated correctly](https://github.com/code-423n4/2023-01-astaria-findings/issues/435) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L597-L609
https://github.com/code-423n4/2023-01-astaria/blob/main/src/LienToken.sol#L819


# Vulnerability details

## Impact
Strategist interest reward is not calculated correctly. The reward will almost always be calculated with interestOwed, regardless of the amount paid.

As a result, the strategist gets paid more than they are supposed to. Even if the borrower hasn't made a single payment, the strategist can make a tiny payment on behalf of the borrower to trigger this calculation.

This also encourages the strategist to maximize interestOwed and make tiny payments on behalf of the borrower. This can trigger a compound interest vulnerability which I've made a separate report about.

## Proof of Concept
As confirmed by the sponsor, the strategist reward is supposed to be "paid on performance and only@on interest. if the payment that’s being made is greater than the interest owing we only mint them based on the interest owed, but if it’s less, then, mint their shares based on the amount"

However, this is not the case. When LienToken calls beforePayment, which calls _handleStrategistInterestReward, the amount passed in is the amount of the lien (stack.point.amount), not the amount paid.

## Tools Used
VSCode

## Recommended Mitigation Steps
https://github.com/code-423n4/2023-01-astaria/blob/main/src/LienToken.sol#L819
I believe `stack.point.amount` should be changed to `amount`.
