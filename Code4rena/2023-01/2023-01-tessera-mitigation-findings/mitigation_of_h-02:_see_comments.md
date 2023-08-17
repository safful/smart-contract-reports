## Tags

- 3 (High Risk)
- primary issue
- selected for report
- sponsor acknowledged
- MR-H-02

# [Mitigation of H-02: See comments](https://github.com/code-423n4/2023-01-tessera-mitigation-findings/issues/21) 

# Lines of code

https://github.com/fractional-company/modular-fractional/blob/fc15c85069d8d55cfe840d4c313754c77f18979f/src/modules/GroupBuy.sol#L198-L199


# Vulnerability details

The [PR](https://github.com/fractional-company/modular-fractional/pull/207) applies the recommended mitigation from the [finding](https://github.com/code-423n4/2022-12-tessera-findings/issues/7), but doesn't take into account the rounding issue identified in [M-09](https://github.com/code-423n4/2022-12-tessera-findings/issues/49) 


## Impact

If the price the NFT is bought for is not an exact multiple of the `filledQuantities`, there will be a loss of precision, and during the [calculation](https://github.com/fractional-company/modular-fractional/blob/fc15c85069d8d55cfe840d4c313754c77f18979f/src/modules/GroupBuy.sol#L260) of refunds, each user will get back more than they should, which will leave the last user with not enough funds to withdraw. It is much more likely for the price _not_ to be an exact multiple than it is for it to be one, so most `GroupBuy`s will have this issue.


## Proof of Concept

The division will have a loss of precision:
```solidity
        // Calculates actual min reserve price based on purchase price
        minReservePrices[_poolId] = _price / filledQuantity;
```
https://github.com/fractional-company/modular-fractional/blob/fc15c85069d8d55cfe840d4c313754c77f18979f/src/modules/GroupBuy.sol#L198-L199

And the [same example](https://github.com/code-423n4/2022-12-tessera-findings/issues/49#issuecomment-1358963067) the judge used in finding M-9 would still apply


## Tools Used

Code inspection


## Recommended Mitigation Steps

Decrease the total number of Raes if the price is not a multiple of the current total, so that it becomes a multiple
