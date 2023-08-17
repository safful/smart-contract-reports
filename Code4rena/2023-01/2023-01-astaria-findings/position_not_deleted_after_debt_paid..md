## Tags

- bug
- 2 (Med Risk)
- downgraded by judge
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-17

# [Position not deleted after debt paid.](https://github.com/code-423n4/2023-01-astaria-findings/issues/343) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/main/src/LienToken.sol#L646


# Vulnerability details

In `_paymentAH` function from LienToken.sol, the `stack` argument should be storage instead of memory. This bug was also disclosed in the Spearbit audit of this program and was resolved during here: https://github.com/AstariaXYZ/astaria-core/pull/201/commits/5a0a86837c0dcf2f6768e8a42aa4215666b57f11, but was later re-introduced https://github.com/AstariaXYZ/astaria-core/commit/be9a14d08caafe125c44f6876ebb4f28f06d83d4 here. Marking it as high-severity as it was marked as same in the audit. 

### POC

```solidity
function _paymentAH(
    LienStorage storage s,
    address token,
    AuctionStack[] memory stack,
    uint256 position,
    uint256 payment,
    address payer
  ) internal returns (uint256) {
    uint256 lienId = stack[position].lienId;
    uint256 end = stack[position].end;
    uint256 owing = stack[position].amountOwed;
    //...[deleted lines to show bug]
    
    delete s.lienMeta[lienId]; //full delete
    delete stack[position]; // <- no effect on storage
    
  }
```

The position here is not deleted after the debt is paid as it is a memory pointer. 

### Recommendation

The first fix can be used again and it would work.