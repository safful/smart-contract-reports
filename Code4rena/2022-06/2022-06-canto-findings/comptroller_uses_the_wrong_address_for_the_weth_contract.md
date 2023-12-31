## Tags

- bug
- 3 (High Risk)
- sponsor confirmed

# [Comptroller uses the wrong address for the WETH contract](https://github.com/code-423n4/2022-06-canto-findings/issues/46) 

# Lines of code

https://github.com/Plex-Engineer/lending-market/blob/755424c1f9ab3f9f0408443e6606f94e4f08a990/contracts/Comptroller.sol#L1469


# Vulnerability details

## Impact
The Comptroller contract uses a hardcoded address for the WETH contract which is not the correct one. Because of that, it will be impossible to claim COMP rewards. That results in a loss of funds so I rate it as HIGH.

## Proof of Concept
The Comptroller's `getWETHAddress()` function: https://github.com/Plex-Engineer/lending-market/blob/755424c1f9ab3f9f0408443e6606f94e4f08a990/contracts/Comptroller.sol#L1469

It's a left-over from the original compound repo: https://github.com/compound-finance/compound-protocol/blob/master/contracts/Comptroller.sol#L1469

It's used by the `grantCompInternal()` function: https://github.com/Plex-Engineer/lending-market/blob/755424c1f9ab3f9f0408443e6606f94e4f08a990/contracts/Comptroller.sol#L1377

That function is called by `claimComp()`: https://github.com/Plex-Engineer/lending-market/blob/755424c1f9ab3f9f0408443e6606f94e4f08a990/contracts/Comptroller.sol#L1365 

If there is a contract stored in that address and it doesn't adhere to the interface (doesn't have a `balanceOf()` and `transfer()` function), the transaction will revert. If there is no contract, the call will succeed without having any effect. In both cases, the user doesn't get their COMP rewards.

## Tools Used
none

## Recommended Mitigation Steps
The WETH contract's address should be parsed to the Comptroller through the constructor or another function instead of being hardcoded.

