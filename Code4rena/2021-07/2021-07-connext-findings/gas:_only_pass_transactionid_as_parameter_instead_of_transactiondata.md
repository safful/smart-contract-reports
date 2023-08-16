## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [Gas: Only pass transactionId as parameter instead of TransactionData](https://github.com/code-423n4/2021-07-connext-findings/issues/19) 

# Handle

cmichel


# Vulnerability details

Both the `recoverFulfillSignature` and `recoverCancelSignature` functions take a large `TransactionData` object as their first argument but only use the `transactionId` field of the struct.
It should be more efficient to only pass `txData.transactionId` as the parameter.

