## Tags

- bug
- 1 (Low Risk)
- sponsor confirmed

# [Add nonreentrant modifiers to external methods in 2 files](https://github.com/code-423n4/2021-09-defiprotocol-findings/issues/270) 

# Handle

tensors


# Vulnerability details

I recommend adding reentrancy checks throughout Basket.sol and Auction.sol
using a mutex lock. Many external calls are made to potentially unsafe token contracts.
In the case that not all token contracts are properly vetted, this preventative step could be worthwhile.

