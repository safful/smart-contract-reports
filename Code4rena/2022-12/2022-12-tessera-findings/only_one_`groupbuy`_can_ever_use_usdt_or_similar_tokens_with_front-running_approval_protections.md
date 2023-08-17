## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- sponsor disputed
- M-06

# [Only one `GroupBuy` can ever use USDT or similar tokens with front-running approval protections](https://github.com/code-423n4/2022-12-tessera-findings/issues/37) 

# Lines of code

https://github.com/code-423n4/2022-12-tessera/blob/f37a11407da2af844bbfe868e1422e3665a5f8e4/src/seaport/targets/SeaportLister.sol#L29-L46


# Vulnerability details

Calling `approve()` without first calling `approve(0)` if the current approval is non-zero will revert with some tokens, such as Tether (USDT). While Tether is known to do this, it applies to other tokens as well, which are trying to protect against [this attack vector](https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit).

## Impact
Only the first listing will start with the conduit's approval at zero and will be able to change it to the maximum. Thereafter, every attempt to approve that same token will revert, causing any order using this lister to revert, including a re-listing at a lower price, which the protocol allows for.

## Proof of Concept
The seaport conduit address is set in the constructor as an immutable variable, so it's not possible to change it once the issue is hit:
```solidity
// File: src/seaport/targets/SeaportLister.sol : SeaportLister.constructor()   #1

19        constructor(address _conduit) {
20 @>         conduit = _conduit;
21:       }
```
https://github.com/code-423n4/2022-12-tessera/blob/f37a11407da2af844bbfe868e1422e3665a5f8e4/src/seaport/targets/SeaportLister.sol#L19-L21

The approval is set to the maximum without check whether it's already the maximum:
```solidity
// File: src/seaport/targets/SeaportLister.sol : SeaportLister.validateListing()   #2

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

The README states: `The Tessera Protocol is designed around the concept of Hyperstructures, which are crypto protocols that can run for free and forever, without maintenance, interruption or intermediaries`, and having to deploy a new `SeaportLister` in order to have a fresh conduit that also needs to be deployed, is not `without maintenance or interruption`.

## Tools Used
Code inspection

## Recommended Mitigation Steps
Always reset the approval to zero before changing it to the maximum
