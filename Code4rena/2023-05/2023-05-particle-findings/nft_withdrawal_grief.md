## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- M-01

# [NFT withdrawal grief](https://github.com/code-423n4/2023-05-particle-findings/issues/44) 

# Lines of code

https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L183


# Vulnerability details



## Impact
A lienee whose NFT is not currently on loan may be prevented from withdrawing it.

## Proof of Concept
A lienee who wishes to withdraw his NFT calls `withdrawNftWithInterest()` which tries to [`IERC721.safeTransferFrom()` the NFT](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L183), which therefore reverts if the NFT is not in the contract (being on loan).
A griefer might therefore sandwich his call to `withdrawNftWithInterest()` with a `swapWithEth()` and a `repayWithNft()`.
[`swapWithEth()` removes the NFT from the contract](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L460), which causes the following `withdrawNftWithInterest()` to revert. For this the griefer has to [pay `lien.credit + lien.price`](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L440). But this is [returned in full in `repayWithNft()`](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L509-L512) [minus `payableInterest`](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/protocol/ParticleExchange.sol#L502) which is [nothing since the loan time is zero](https://github.com/code-423n4/2023-05-particle/blob/1caf678bc20c24c96fc8f6b0046383ff0e9d2a6f/contracts/libraries/math/MathUtils.sol#L24-L25).

## Recommended Mitigation Steps
Allow the option to, at any time, set a lien to not accept a new loan.


## Assessed type

DoS