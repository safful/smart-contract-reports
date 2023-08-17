## Tags

- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- MR-NEW

# [StRSR: attacker can steal excess rsr that is returned after seizure](https://github.com/code-423n4/2023-02-reserve-mitigation-contest-findings/issues/17) 

# Lines of code

https://github.com/reserve-protocol/protocol/blob/27a3472d553b4fa54f896596007765ec91941348/contracts/p1/BackingManager.sol#L176-L182
https://github.com/reserve-protocol/protocol/blob/27a3472d553b4fa54f896596007765ec91941348/contracts/p1/StRSR.sol#L496-L530


# Vulnerability details

**Note:**  
This issue deals with excess `rsr` that was seized from `stRSR` but is returned again.  
The `M-12` issue also deals with excess `rsr`.  

However `M-12` deals with the fact that not all `rsr` is returned to `stRSR`, whereas this issue deals with the fact that an attacker can steal `rsr` once it is returned to `stRSR`.  

So while the issues seem to be similar they in fact are different.  

They are separate issues. So I chose to report this separately with the `NEW` keyword.  

## Impact
`rsr` can be returned to `stRSR` after a seizure if not all seized `rsr` has been necessary to regain full collateralization.  

This happens in the [`BackingManger.handoutExcessAssets`](https://github.com/reserve-protocol/protocol/blob/27a3472d553b4fa54f896596007765ec91941348/contracts/p1/BackingManager.sol#L176-L182) function.  

This excess `rsr` is then paid out to ALL stakers just like regular `rsr` rewards using the [`StRSR._payoutRewards`](https://github.com/reserve-protocol/protocol/blob/27a3472d553b4fa54f896596007765ec91941348/contracts/p1/StRSR.sol#L496-L530) function.  

This is unfair. An attacker can abuse this behavior and stake `rsr` to profit from the returned `rsr` which is used to appreciate his `stRSR`.  

It would be fair if the `rsr` was returned only to the users that had staked when the seizure occurred.  

## Proof of Concept
Think of the following scenario:  

1. There are currently 100 stakers with an equal share of the `1000 rsr` total that is currently in the `stRSR` contract.  
2. A seizure occurs and `500 rsr` are seized.
3. Not all `rsr` is sold and some (say `50 rsr`) is returned to `StRSR`
4. The attacker can front-run the transaction that returns the `rsr` and become a staker himself
5. The attacker will profit from the returned `rsr` once it is paid out as reward. Say the attacker stakes `100 rsr`. He now owns a share of `100 rsr / (500 rsr + 100 rsr) = 20%`. This means he will also get `20%` of the `50 rsr` that are paid out as rewards.  

## Tools Used
VSCode

## Recommended Mitigation Steps
Ideally, as I said above, the `rsr` should be returned only to the users that had staked when the seizure occurred.  

With the current architecture of the `stRSR` contract this is not possible. There is no way to differentiate between stakers.  

Also the scenario described is an edge and relies on a seizure to occur and `rsr` to be returned.  

It seems unrealistic that `10%` of the seized `rsr` is returned again. I think a number like `1% - 5%` is more realistic.  

But still if the amount of `rsr` that is seized is big enough, `1% - 5%` can be a significant amount in terms of dollar value.  

I estimate this to be `Medium` severity since an attacker can profit at the expense of other stakers and this behavior will decrease the willingness of users to stake as the risk of losing funds is increased.  

This severly damages the incentives involved with staking. Stakers are incentivized to wait for seizures to occur and only then stake as they might profit from returned `rsr`.  

I encourage the sponsor to further assess if there is a better way to return excess `rsr`.  