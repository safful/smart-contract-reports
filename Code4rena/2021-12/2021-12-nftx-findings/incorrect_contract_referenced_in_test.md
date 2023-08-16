## Tags

- bug
- 0 (Non-critical)
- resolved
- sponsor confirmed

# [Incorrect contract referenced in test](https://github.com/code-423n4/2021-12-nftx-findings/issues/116) 

# Handle

sirhashalot


# Vulnerability details

## Impact

The file contracts/solidity/testing/NFTXFeeDistributor2.sol references the old NFTXFeeDistributor.sol and instead should reference the new NFTXSimpleFeeDistributor.sol

## Proof of Concept

The import and contract inheritance of contracts/solidity/testing/NFTXFeeDistributor2.sol

## Tools Used

`npx hardhat test` fails due to this issue because the ../NFTXFeeDistributor.sol imported file is not found

## Recommended Mitigation Steps

Reference the new NFTXSimpleFeeDistributor.sol and not NFTXFeeDistributor.sol in the import and contract inheritance

