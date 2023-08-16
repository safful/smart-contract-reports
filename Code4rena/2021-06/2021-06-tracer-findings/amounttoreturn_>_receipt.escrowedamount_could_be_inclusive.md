## Tags

- bug
- G (Gas Optimization)
- sponsor confirmed

# [amountToReturn > receipt.escrowedAmount could be inclusive](https://github.com/code-423n4/2021-06-tracer-findings/issues/108) 

# Handle

pauliax


# Vulnerability details

## Impact
Could save some gas here when amountToReturn = receipt.escrowedAmount:
    if (amountToReturn > receipt.escrowedAmount) {
       liquidationReceipts[receiptId].escrowedAmount = 0;
    } else {
       liquidationReceipts[receiptId].escrowedAmount = receipt.escrowedAmount - amountToReturn;
    }

## Recommended Mitigation Steps
 if (amountToReturn >= receipt.escrowedAmount) {
...

