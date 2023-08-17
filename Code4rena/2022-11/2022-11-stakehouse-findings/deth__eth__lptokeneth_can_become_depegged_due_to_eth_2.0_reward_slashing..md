## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- M-16

# [dETH / ETH / LPTokenETH can become depegged due to ETH 2.0 reward slashing.](https://github.com/code-423n4/2022-11-stakehouse-findings/issues/164) 

# Lines of code

https://github.com/code-423n4/2022-11-stakehouse/blob/4b6828e9c807f2f7c569e6d721ca1289f7cf7112/contracts/liquid-staking/GiantSavETHVaultPool.sol#L66


# Vulnerability details

## Impact

dETH / ETH / LPTokenETH can become depegged due to ETH 2.0 reward slashing.

## Proof of Concept

I want to quote the info from the doc:

> SavETH Vault - users can pool up to `24 ETH` where protected staking ensures no-loss. dETH can be redeemed after staking

and

> Allocate savETH <> dETH to `savETH Vault` (24 dETH)

However, the main risk in ETH 2.0 POS staking is the slashing penalty, in that case the ETH will not be pegged and the validator cannot maintain a minimum 32 ETH staking balance.

https://cryptobriefing.com/ethereum-2-0-validators-slashed-staking-pool-error/

## Tools Used

Manual Review

## Recommended Mitigation Steps

We recommand the protocol to add mechanism to ensure the dETH is pegged via burning if case the ETH got slashed.

and consider when the node do not maintain a minmum 32 ETH staking balance, who is charge of adding the ETH balance to increase

the staking balance or withdraw the ETH and distribute the fund.