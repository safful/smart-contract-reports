## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor acknowledged
- M-05

# [Orders may not be fillable due to missing approvals](https://github.com/code-423n4/2022-12-tessera-findings/issues/36) 

# Lines of code

https://github.com/code-423n4/2022-12-tessera/blob/f37a11407da2af844bbfe868e1422e3665a5f8e4/src/seaport/targets/SeaportLister.sol#L40


# Vulnerability details

Not all `IERC20` implementations `revert()` when there's a failure in `approve()`. If one of these tokens returns false, there is no check for whether this has happened during the order listing validation, so it will only be detected when the order is attempted. 

## Impact
If the approval failure isn't detected, the listing will never be fillable, because the funds won't be able to be pulled from the opensea conduit. Once this happens, and if it's detected, the only way to fix it is to create a counter-listing at a lower price (which may be below the market value of the tokens), waiting for the order to expire (which it may never), or by buying out all of the Rae to cancel the order (very expensive and defeats the purpose of pooling funds in the first place).

## Proof of Concept
The return value of `approve()` isn't checked, so the order will be allowed to be listed without having approved the conduit:
```solidity
// File: src/seaport/targets/SeaportLister.sol : SeaportLister.validateListing()   #1

29                for (uint256 i; i < ordersLength; ++i) {
30                    uint256 offerLength = _orders[i].parameters.offer.length;
31                    for (uint256 j; j < offerLength; ++j) {
32                        OfferItem memory offer = _orders[i].parameters.offer[j];
33                        address token = offer.token;
34                        ItemType itemType = offer.itemType;
35                        if (itemType == ItemType.ERC721)
36                            IERC721(token).setApprovalForAll(conduit, true);
37                        if (itemType == ItemType.ERC1155)
38                            IERC1155(token).setApprovalForAll(conduit, true);
39                        if (itemType == ItemType.ERC20)
40 @>                         IERC20(token).approve(conduit, type(uint256).max);
41                    }
42                }
43            }
44            // Validates the order on-chain so no signature is required to fill it
45            assert(ConsiderationInterface(_consideration).validate(_orders));
46:       }
```
https://github.com/code-423n4/2022-12-tessera/blob/f37a11407da2af844bbfe868e1422e3665a5f8e4/src/seaport/targets/SeaportLister.sol#L29-L46

## Tools Used
Code inspection

## Recommended Mitigation Steps
Use OpenZeppelin's `safeApprove()`, which checks the return code and reverts if it's not success
