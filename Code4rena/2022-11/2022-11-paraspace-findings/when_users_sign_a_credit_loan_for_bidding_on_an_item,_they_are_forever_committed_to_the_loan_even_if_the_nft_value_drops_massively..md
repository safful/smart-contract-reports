## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- M-16

# [When users sign a credit loan for bidding on an item, they are forever committed to the loan even if the NFT value drops massively.](https://github.com/code-423n4/2022-11-paraspace-findings/issues/474) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/libraries/types/DataTypes.sol#L296


# Vulnerability details

## Description

In ParaSpace marketplace, taker may pass maker's signature and fulfil their bid with taker's NFT. The maker can use credit loan to purchase the NFT provided the health factor is positive in the end.

In validateAcceptBidWithCredit, verifyCreditSignature  is called to verify maker signed the credit structure.

```
function verifyCreditSignature(
    DataTypes.Credit memory credit,
    address signer,
    uint8 v,
    bytes32 r,
    bytes32 s
) private view returns (bool) {
    return
        SignatureChecker.verify(
            hashCredit(credit),
            signer,
            v,
            r,
            s,
            getDomainSeparator()
        );
}
```

The issue is that the credit structure does not have a deadline:
```
struct Credit {
    address token;
    uint256 amount;
    bytes orderId;
    uint8 v;
    bytes32 r;
    bytes32 s;
}
```

As a result, attacker may simply wait and if the price of the NFT goes down abuse their previous signature to take a larger amount than they would like to pay for the NFT. Additionaly, there is no revocation mechanism, so user has completely committed to loan to get the NFT until the end of time.

## Impact

When users sign a credit loan for bidding on an item, they are forever committed to the loan even if the NFT value drops massively.


## Tools Used

Manual audit

## Recommended Mitigation Steps

Add a deadline timestamp to the signed credit structure.