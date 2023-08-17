## Tags

- bug
- disagree with severity
- downgraded by judge
- grade-a
- primary issue
- QA (Quality Assurance)
- selected for report
- edited-by-warden
- Q-25

# [computePhase0Eth1DataRoot always returns an incorrect Merkle tree](https://github.com/code-423n4/2023-04-eigenlayer-findings/issues/63) 

# Lines of code

https://github.com/Layr-Labs/eigenlayer-contracts/blob/eccdfd43bb882d66a68cad8875dde2979e204546/src/contracts/libraries/BeaconChainProofs.sol#L160


# Vulnerability details

## Impact
Detailed description of the impact of this finding.
The Merkle tree creation inside the computePhase0Eth1DataRoot function is incorrect
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
Not all fields of eth1DataFields being used in an array due to usage of `i < ETH1_DATA_FIELD_TREE_HEIGHT` instead of `i<NUM_ETH1_DATA_FIELDS`
Check other similar function.
```solidity
    function computePhase0Eth1DataRoot(bytes32[NUM_ETH1_DATA_FIELDS] calldata eth1DataFields) internal pure returns(bytes32) {  
        bytes32[] memory paddedEth1DataFields = new bytes32[](2**ETH1_DATA_FIELD_TREE_HEIGHT);
        
        for (uint256 i = 0; i < ETH1_DATA_FIELD_TREE_HEIGHT; ++i) {
            paddedEth1DataFields[i] = eth1DataFields[i];
        }

        return Merkle.merkleizeSha256(paddedEth1DataFields);
    }

```
[src/contracts/libraries/BeaconChainProofs.sol#L160](https://github.com/Layr-Labs/eigenlayer-contracts/blob/eccdfd43bb882d66a68cad8875dde2979e204546/src/contracts/libraries/BeaconChainProofs.sol#L160)
## Tools Used
Manual
## Recommended Mitigation Steps

```diff
    function computePhase0Eth1DataRoot(bytes32[NUM_ETH1_DATA_FIELDS] calldata eth1DataFields) internal pure returns(bytes32) {  
        bytes32[] memory paddedEth1DataFields = new bytes32[](2**ETH1_DATA_FIELD_TREE_HEIGHT);
        
_        for (uint256 i = 0; i < ETH1_DATA_FIELD_TREE_HEIGHT; ++i) {
+        for (uint256 i = 0; i < NUM_ETH1_DATA_FIELDS; ++i) {
           paddedEth1DataFields[i] = eth1DataFields[i];
        }

        return Merkle.merkleizeSha256(paddedEth1DataFields);
    }

```


## Assessed type

Math