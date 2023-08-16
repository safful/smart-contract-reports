## Tags

- bug
- 0 (Non-critical)
- sponsor confirmed

# [TRIBERagequit: Make verifyClaim() public](https://github.com/code-423n4/2021-11-fei-findings/issues/107) 

# Handle

hickuphh3


# Vulnerability details

## Suggestion

Based on past experiences with on-chain actions involving merkle proofs, users tend to ask for the merkle tree data so that they can, in this case, ragequit their TRIBE for FEI by directly interacting with the contract on Etherscan.

It would therefore be helpful for `verifyClaim()` to be made public for users to verify that the merkle proof they input via Etherscan is correct.

