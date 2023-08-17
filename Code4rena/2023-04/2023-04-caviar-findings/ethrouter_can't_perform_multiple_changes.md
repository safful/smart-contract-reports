## Tags

- bug
- 2 (Med Risk)
- high quality report
- primary issue
- selected for report
- sponsor confirmed
- M-02

# [EthRouter can't perform multiple changes](https://github.com/code-423n4/2023-04-caviar-findings/issues/873) 

# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/cd8a92667bcb6657f70657183769c244d04c015c/src/EthRouter.sol#L273


# Vulnerability details

## Impact
EthRouter is meant to support multiple changes in one tx, but that would fail

## Proof of Concept

The function `EthRouter.change` sends `msg.value` to pool in a for loop:

```js
for (uint256 i = 0; i < changes.length; i++) {
    Change memory _change = changes[i];

    ...

    // execute change
    PrivatePool(_change.pool).change{value: msg.value}(
        _change.inputTokenIds,
        _change.inputTokenWeights,
        _change.inputProof,
        _change.stolenNftProofs,
        _change.outputTokenIds,
        _change.outputTokenWeights,
        _change.outputProof
    );
```
The pool subtracts the fee, and sends the rest back to the router. After the first iteration the router contains less ETH than `msg.value` and will revert

<details>
  <summary>POC here</summary>
  
Add to `Change.t.sol` and run with `forge test --match test_twoChanges -vvvv`

```js
    function test_twoChangesOneCall() public {
    uint256[] memory inputTokenIds = new uint256[](1);
    uint256[] memory inputTokenWeights = new uint256[](0);
    uint256[] memory outputTokenIds = new uint256[](1);
    uint256[] memory outputTokenWeights = new uint256[](0);

    uint256[] memory inputTokenIds2 = new uint256[](1);
    uint256[] memory inputTokenWeights2 = new uint256[](0);
    uint256[] memory outputTokenIds2 = new uint256[](1);
    uint256[] memory outputTokenWeights2 = new uint256[](0);

    inputTokenIds[0] = 5;
    outputTokenIds[0] = 0;

    inputTokenIds2[0] = 6;
    outputTokenIds2[0] = 1;

    EthRouter.Change[] memory changes = new EthRouter.Change[](2);
    changes[0] = EthRouter.Change({
        pool: payable(address(privatePool)),
        nft: address(milady),
        inputTokenIds: inputTokenIds,
        inputTokenWeights: inputTokenWeights,
        inputProof: PrivatePool.MerkleMultiProof(new bytes32[](0), new bool[](0)),
        stolenNftProofs: new IStolenNftOracle.Message[](0),
        outputTokenIds: outputTokenIds,
        outputTokenWeights: outputTokenWeights,
        outputProof: PrivatePool.MerkleMultiProof(new bytes32[](0), new bool[](0))
    });

    changes[1] = EthRouter.Change({
        pool: payable(address(privatePool)),
        nft: address(milady),
        inputTokenIds: inputTokenIds2,
        inputTokenWeights: inputTokenWeights2,
        inputProof: PrivatePool.MerkleMultiProof(new bytes32[](0), new bool[](0)),
        stolenNftProofs: new IStolenNftOracle.Message[](0),
        outputTokenIds: outputTokenIds2,
        outputTokenWeights: outputTokenWeights2,
        outputProof: PrivatePool.MerkleMultiProof(new bytes32[](0), new bool[](0))
    });

    (uint256 changeFee,) = privatePool.changeFeeQuote(inputTokenIds.length * 1e18);

    //WARDEN: multiply with 10 just to make sure there really is enough
    ethRouter.change{value: changeFee*10}(changes, 0);

} 
  ```
  
Output:

```
...
    │   ├─ [0] PrivatePool::change{value: 50000000000000000000}([6], [], ([], []), [], [1], [], ([], [])) 
    │   │   └─ ← "EvmError: OutOfFund"
    │   └─ ← "EvmError: Revert"
    └─ ← "EvmError: Revert"
```

</details>

## Tools Used

Manual review

## Recommended Mitigation Steps
only send the required change fee and not `msg.value`