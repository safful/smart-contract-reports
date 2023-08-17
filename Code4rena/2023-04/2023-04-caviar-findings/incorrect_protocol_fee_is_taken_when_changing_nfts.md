## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- sponsor confirmed
- M-10

# [Incorrect protocol fee is taken when changing NFTs](https://github.com/code-423n4/2023-04-caviar-findings/issues/463) 

# Lines of code

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L737


# Vulnerability details

## Impact
Incorrect protocol fee is taken when changing NFTs which results in profit loss for the Caviar protocol.

## Proof of Concept
The protocol fee in changeFeeQuote is calculated as a percentage of the feeAmount which is based on the input amount:

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L737
```solidity
function changeFeeQuote(uint256 inputAmount) public view returns (uint256 feeAmount, uint256 protocolFeeAmount) {
    ...
    protocolFeeAmount = feeAmount * Factory(factory).protocolFeeRate() / 10_000;
```

This seems wrong as in buyQuote and sellQuote the protocol fee is calculated as a percentage of the input amount, not the pool fee amount:

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L703
```solidity
function buyQuote(uint256 outputAmount)
    ...
    protocolFeeAmount = inputAmount * Factory(factory).protocolFeeRate() / 10_000;
```

https://github.com/code-423n4/2023-04-caviar/blob/main/src/PrivatePool.sol#L721
```solidity
function sellQuote(uint256 inputAmount)
    ...
    protocolFeeAmount = outputAmount * Factory(factory).protocolFeeRate() / 10_000;
```

This makes the protocol fee extremely low meaning a profit loss for the protocol.

## Tools Used
Manual review

## Recommended Mitigation Steps
protocolFeeAmount in changeFeeQuote should be a percentage of the input amount instead of the pool fee.