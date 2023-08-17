## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- M-04

# [Iterations over all tiers in recordMintBestAvailableTier can render system unusable](https://github.com/code-423n4/2022-10-juicebox-findings/issues/64) 

# Lines of code

https://github.com/jbx-protocol/juice-nft-rewards/blob/89cea0e2a942a9dc9e8d98ae2c5f1b8f4d916438/contracts/JBTiered721DelegateStore.sol#L950


# Vulnerability details

## Impact
`JBTiered721DelegateStore.recordMintBestAvailableTier` potentially iterates over all tiers to find the one with the highest contribution floor that is lower than `_amount`. When there are many tiers, this loop can always run out of gas, which will cause some transactions (the ones that have a high `_leftoverAmount` within `_processPayment`) to always revert. The (implicit) limit for the number of tiers is 2^16 - 1, so it is possible that this happens in practice.

## Proof Of Concept
Let's say that 1,000 tiers are registered for a project. Small payments without a leftover amount or a small amount will be succesfully processed by `_processPayment`, because `_mintBestAvailableTier` is either not called or it is called with a small amount, meaning that `recordMintBestAvailableTier` will exit the loop early (when it is called with a small amount). However, if a payment with a large leftover amount (let's say greater than the highest contribution floor) is processed, it is necessary to iterate over all tiers, which will use too much gas and cause the processing to revert.

## Recommended Mitigation Steps
Use a binary search (which requires some architectural changes) for determining the best available tier. Then, the gas usage grows logarithmically (instead of linear with the current design) with the number of tiers, meaning that it would only be ~16 times higher for 65535 tiers as for 2 tiers.