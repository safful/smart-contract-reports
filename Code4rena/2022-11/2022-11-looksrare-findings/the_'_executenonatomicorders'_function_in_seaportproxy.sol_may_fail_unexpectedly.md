## Tags

- bug
- 2 (Med Risk)
- satisfactory
- selected for report
- sponsor acknowledged
- edited-by-warden
- M-01

# [The '_executeNonAtomicOrders' function in SeaportProxy.sol may fail unexpectedly](https://github.com/code-423n4/2022-11-looksrare-findings/issues/54) 

# Lines of code

https://github.com/code-423n4/2022-11-looksrare/blob/e3b2c053f722b0ca2dce3a3eb06f64859b8b7a6f/contracts/proxies/SeaportProxy.sol#L224


# Vulnerability details

## Impact
The '_executeNonAtomicOrders' function in SeaportProxy.sol tries to send fee by batch, this may break the 'NonAtomic' feature.

## Proof of Concept
Let's say user want to buy 3 NFTs with following order parameters
```
NFT_1: price = 100 USDT, fee = 5%
NFT_2: price = 200 USDT, fee = 5%
NFT_3: price = 300 USDT, fee = 5%
```
Given user only sends 600 USDT, the expected result should be
```
NFT_1: success
NFT_2: success
NFT_3: failed
```
But as the fees are batched and sent at last, cause all 3 orders failed.

```
function _executeNonAtomicOrders(
    BasicOrder[] calldata orders,
    bytes[] calldata ordersExtraData,
    address recipient,
    uint256 feeBp,
    address feeRecipient
) private {
    uint256 fee;
    address lastOrderCurrency;
    for (uint256 i; i < orders.length; ) {
        OrderExtraData memory orderExtraData = abi.decode(ordersExtraData[i], (OrderExtraData));
        AdvancedOrder memory advancedOrder;
        advancedOrder.parameters = _populateParameters(orders[i], orderExtraData);
        advancedOrder.numerator = orderExtraData.numerator;
        advancedOrder.denominator = orderExtraData.denominator;
        advancedOrder.signature = orders[i].signature;

        address currency = orders[i].currency;
        uint256 price = orders[i].price;

        try
            marketplace.fulfillAdvancedOrder{value: currency == address(0) ? price : 0}(
                advancedOrder,
                new CriteriaResolver[](0),
                bytes32(0),
                recipient
            )
        {
            if (feeRecipient != address(0)) {
                uint256 orderFee = (price * feeBp) / 10000;
                if (currency == lastOrderCurrency) {
                    fee += orderFee;
                } else {
                    if (fee > 0) _transferFee(fee, lastOrderCurrency, feeRecipient);

                    lastOrderCurrency = currency;
                    fee = orderFee;
                }
            }
        } catch {}

        unchecked {
            ++i;
        }
    }

    if (fee > 0) _transferFee(fee, lastOrderCurrency, feeRecipient);
}
```


## Tools Used
VS Code

## Recommended Mitigation Steps
Don't batch up fees.