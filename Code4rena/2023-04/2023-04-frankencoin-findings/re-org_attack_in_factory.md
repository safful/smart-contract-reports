## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- edited-by-warden
- M-14

# [Re-org attack in factory](https://github.com/code-423n4/2023-04-frankencoin-findings/issues/155) 

# Lines of code

https://github.com/code-423n4/2023-04-frankencoin/blob/1022cb106919fba963a89205d3b90bf62543f68f/contracts/PositionFactory.sol#L44


# Vulnerability details

## Impact

The `createClone` function deploys a clone contract using the create, where the address derivation depends only on the `PositionFactory` nonce.

Re-orgs can happen in all EVM chains. In ethereum, where currently Frankencoin is deployed, it is not "super common" but it still happens, being the last one less than a year ago: 

https://decrypt.co/101390/ethereum-beacon-chain-blockchain-reorg

The issue increases the changes of happening because frankencoin is thinking about deploying also in L2's/ rollups, proof:

https://discord.com/channels/810916927919620096/1095308824354758696/1096693817450692658

where re-orgs have been much more active: 

https://protos.com/polygon-hit-by-157-block-reorg-despite-hard-fork-to-reduce-reorgs/

being the last one, less than a year ago.

The issue would happen when users rely on the address derivation in advance or try to deploy the position clone with the same address on different EVM chains, any funds sent to the new clone could potentially be withdrawn by anyone else. All in all, it could lead to the theft of user funds.

As you can see in a previous report, the issue should be marked and judged as a medium: 

https://code4rena.com/reports/2023-01-rabbithole/#m-01-questfactory-is-suspicious-of-the-reorg-attack

## Proof of Concept

Imagine that Alice deploys a position clone, and then sends funds to it. Bob sees that the network block reorg happens and calls `clonePosition`. Thus, it creates a position clone with an address to which Alice sends funds. Then Alice's transactions are executed and Alice transfers funds to Bob’s position contract.

## Tools Used

Manual

## Recommended Mitigation Steps

The recommendation is basically the same as: 

https://code4rena.com/reports/2023-01-rabbithole/#m-01-questfactory-is-suspicious-of-the-reorg-attack

Deploy the cloned Position via create2 with a specific salt that includes msg.sender and address `_existing`
