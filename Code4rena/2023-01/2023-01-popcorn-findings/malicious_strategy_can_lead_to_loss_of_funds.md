## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- H-05

# [Malicious strategy can lead to loss of funds](https://github.com/code-423n4/2023-01-popcorn-findings/issues/388) 

# Lines of code

https://github.com/code-423n4/2023-01-popcorn/blob/main/src/vault/adapter/abstracts/AdapterBase.sol#L444


# Vulnerability details

## Impact
A malicious strategy has access to the adapter's storage and can therefore freely change any values.

## Proof of Concept

Because `AdapterBase` calls the `Strategy` using `delegatecall`, the `Strategy` has access to the calling contract's storage and can be manipulated directly.

In the following proof of concept, a `MaliciousStrategy` is paired with the `BeefyAdapter` and when called will manipulate the `performanceFee` and `highWaterMark` values. Of course, any other storage slots of the adapter could also be manipulated or any other calls to external contracts on behalf of the `msg.sender` could be performed.

`MaliciousStrategy` implementation showing the exploit -
https://gist.github.com/alpeware/e0b1c9f330419986142711e814bfdc7b#file-beefyadapter-t-sol-L18

`Adapter` helper used to determine the storage slots -
https://gist.github.com/alpeware/e0b1c9f330419986142711e814bfdc7b#file-beefyadapter-t-sol-L65

`BeefyAdapterTest` changes made to tests -

Adding the malicious strategy -
https://gist.github.com/alpeware/e0b1c9f330419986142711e814bfdc7b#file-beefyadapter-t-sol-L123

Adding new test `test__StrategyHarvest()` executing `harvest()` -
https://gist.github.com/alpeware/e0b1c9f330419986142711e814bfdc7b#file-beefyadapter-t-sol-L132

Log output -
https://gist.github.com/alpeware/e0b1c9f330419986142711e814bfdc7b#file-log-txt

## Tools Used

Foundry

## Recommended Mitigation Steps

From chatting with the devs, the goal is to mix and match adapters and strategies. I don't think `delegatecall` should be used and adapters and strategies should be treated as separate contracts. Relevant approvals should be given individually instead.