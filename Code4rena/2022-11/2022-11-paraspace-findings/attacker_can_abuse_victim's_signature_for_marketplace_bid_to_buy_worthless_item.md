## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- M-17

# [Attacker can abuse victim's signature for marketplace bid to buy worthless item](https://github.com/code-423n4/2022-11-paraspace-findings/issues/475) 

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

The issue is that the credit structure does not have a marketplace identifier:
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

As a result, attacker can use the victim's signature for some orderId in a particular marketplace for another one, where this orderId leads to a much lower valued item.
User would borrow money to buy victim's valueless item. This would be HIGH impact, but incidentally right now only the SeaportAdapter marketplace supports credit loans to maker (implements matchBidWithTakerAsk). However, it is very likely the supporting code will be added to LooksRareAdapter and X2Y2Adapter as well. 
LooksRareExchange supports the function out of the box:

```
function matchBidWithTakerAsk(OrderTypes.TakerOrder calldata takerAsk, OrderTypes.MakerOrder calldata makerBid)
    external
    override
    nonReentrant
{
    require((!makerBid.isOrderAsk) && (takerAsk.isOrderAsk), "Order: Wrong sides");
    require(msg.sender == takerAsk.taker, "Order: Taker must be the sender");
    // Check the maker bid order
    bytes32 bidHash = makerBid.hash();
    _validateOrder(makerBid, bidHash);
    (bool isExecutionValid, uint256 tokenId, uint256 amount) = IExecutionStrategy(makerBid.strategy)
        .canExecuteTakerAsk(takerAsk, makerBid);
    require(isExecutionValid, "Strategy: Execution invalid");
    // Update maker bid order status to true (prevents replay)
    _isUserOrderNonceExecutedOrCancelled[makerBid.signer][makerBid.nonce] = true;
    // Execution part 1/2
    ...
}
```

So, this impact would be HIGH but since it is currently not implemented, would downgrade to MED. I understand it can be closed as OOS due to speculation of future code, however I would ask to consider that the likelihood of other Exchanges supporting the required API is particularly high, and take into account the value of this contribution.

## Impact

Attacker can abuse victim's signature for marketplace bid to buy worthless item


## Tools Used

Manual audit

## Recommended Mitigation Steps

Credit structure should contain an additional field "MarketplaceAddress"

