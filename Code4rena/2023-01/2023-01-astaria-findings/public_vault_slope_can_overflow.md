## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- M-10

# [Public vault slope can overflow](https://github.com/code-423n4/2023-01-astaria-findings/issues/418) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/PublicVault.sol#L562-L568
https://github.com/code-423n4/2023-01-astaria/blob/57c2fe33c1d57bc2814bfd23592417fc4d5bf7de/src/LienToken.sol#L702-L704


# Vulnerability details

## Impact
The slope of public vault can overflow in the afterPayment function due to unchecked addition. When this happens, totalAssets will not be correct. This can also result in underflows in slope updates elsewhere, causing large fluctuations in slope and totalAssets.

## Proof of Concept
Assume the token is a normal 18 decimal ERC20 token. 
After 5 loans of 1000 tokens, all with the maximum interest rate of 63419583966, the slope will overflow.
5 * 1000 * 63419583966 / 2^48 = 1.1265581173

## Tools Used
VSCode

## Recommended Mitigation Steps
Remove the unchecked block. Also, I think 48 bits might not be enough for slope.