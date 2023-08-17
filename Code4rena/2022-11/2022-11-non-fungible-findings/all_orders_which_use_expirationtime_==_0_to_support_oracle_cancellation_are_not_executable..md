## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor acknowledged
- M-03

# [All orders which use expirationTime == 0 to support oracle cancellation are not executable.](https://github.com/code-423n4/2022-11-non-fungible-findings/issues/181) 

# Lines of code

https://github.com/code-423n4/2022-11-non-fungible/blob/323b7cbf607425dd81da96c0777c8b12e800305d/contracts/Exchange.sol#L378


# Vulnerability details

## Description

The Non Fungible supplied docs state:
**Off-chain methods**
"Oracle cancellations - if the order is signed with an expirationTime of 0, a user can request an oracle to stop producing authorization signatures; without a recent signature, the order will not be able to be matched"

From the docs, we can expect that when expirationTime is 0 , trader wishes to enable dynamic oracle cancellations. However, the recent code refactoring broke not only this functionality but all orders with expirationTime = 0.

Previous \_validateOrderParameters:
```
function _validateOrderParameters(Order calldata order, bytes32 orderHash)
    internal
    view
    returns (bool)
{
    return (
        /* Order must have a trader. */
        (order.trader != address(0)) &&
        /* Order must not be cancelled or filled. */
        (cancelledOrFilled[orderHash] == false) &&
        /* Order must be settleable. */
        _canSettleOrder(order.listingTime, order.expirationTime)
    );
}
/**
 * @dev Check if the order can be settled at the current timestamp
 * @param listingTime order listing time
 * @param expirationTime order expiration time
 */
function _canSettleOrder(uint256 listingTime, uint256 expirationTime)
    view
    internal
    returns (bool)
{
    return (listingTime < block.timestamp) && (expirationTime == 0 || block.timestamp < expirationTime);
```

New \_validateOrderParameters:
```
function _validateOrderParameters(Order calldata order, bytes32 orderHash)
    internal
    view
    returns (bool)
{
    return (
        /* Order must have a trader. */
        (order.trader != address(0)) &&
        /* Order must not be cancelled or filled. */
        (!cancelledOrFilled[orderHash]) &&
        /* Order must be settleable. */
        (order.listingTime < block.timestamp) &&
        (block.timestamp < order.expirationTime)
    );
}
```

Note the requirements on expirationTime in \_canSettleOrder (old) and \_validateOrderParameters (new).
If expirationTime == 0, the condition was satisfied without looking at `block.timestamp < expirationTime`.
In the new code, `block.timestamp < expirationTime` is *always* required in order for the order to be valid. Clearly, block.timestamp < 0 will always be false, so all orders that wish to make use of off-chain cancellation will never execute.

## Impact

All orders which use expirationTime == 0 to support oracle cancellation are not executable.

## Tools Used

Manual audit, diffing tool

## Recommended Mitigation Steps

Implement the checks the same way as they were in the previous version of Exchange.