## Tags

- bug
- 2 (Med Risk)
- sponsor disputed
- selected for report

# [User fund lost because they can't withdraw() their funds before epoch startTime and they have to stuck in positions that become unprofitable even when epoch is not started](https://github.com/code-423n4/2022-09-y2k-finance-findings/issues/447) 

# Lines of code

https://github.com/code-423n4/2022-09-y2k-finance/blob/2175c044af98509261e4147edeb48e1036773771/src/Vault.sol#L195-L234


# Vulnerability details

## Impact
users deposit their funds in `Vault` when epoch is not started but as other users deposit funds too or price of pegged token changes users get different risk to reward and they may wants to withdraw their funds before epoch start time to get out of bad position but there is no logic in code to give them ability to withdraw their funds before epoch start time.

## Proof of Concept
`Withdraw()` function in `Vault` is only allows users to withdraw after epoch ends and their no logics in contract to allow users to withdraw their funds before epoch start time.
after users deposit their funds the risk to reward ratio of their investment changes as other users deposit funds in one of the Vaults and user may wants to withdraw their funds if they saw that position is bad for them or maybe the price of that token has been changed dramatically before epoch startTime and users wants to withdraw, but there is no functionality that gives users the ability to withdraw their funds before epoch start time and users lose control of their funds after depositing and before epoch start time. as epoch is not started yet users should be able to withdraw their funds but there is not such functionality in the code.

## Tools Used
VIM

## Recommended Mitigation Steps
add some logics to give users ability to withdraw funds before epoch start time.