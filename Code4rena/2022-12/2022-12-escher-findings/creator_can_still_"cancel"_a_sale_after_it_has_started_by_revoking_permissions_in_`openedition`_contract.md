## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- sponsor confirmed
- M-11

# [Creator can still "cancel" a sale after it has started by revoking permissions in `OpenEdition` contract](https://github.com/code-423n4/2022-12-escher-findings/issues/399) 

# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/OpenEdition.sol#L75


# Vulnerability details

The `OpenEdition` type of sale has a start time and an end time. The creator (or owner of the contract) can cancel a sale using the `cancel` function only if it hasn't started yet (i.e. start time is after current block timestamp).

However, the NFT creator can still revoke the minting permissions to the sale contract if he wishes to pull out of the sale. This will prevent anyone from calling the `buy` and prevent any further sale from the collection.

## Impact

The owner of the sale contract can still virtually cancel a sale after it has started by simply revoking the minting permissions to the sale contract.

This will cause the `buy` function to fail because the call to `mint` will revert, effectively making it impossible to further purchase NFTs and causing the effect of canceling the sale.

## PoC

In the following test, the creator of the sale decides to pull from it in the middle of the elapsed duraction. After that, he only needs to wait until the end time passes to call `finalize` and end the sale.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

import "forge-std/Test.sol";
import {FixedPriceFactory} from "src/minters/FixedPriceFactory.sol";
import {FixedPrice} from "src/minters/FixedPrice.sol";
import {OpenEditionFactory} from "src/minters/OpenEditionFactory.sol";
import {OpenEdition} from "src/minters/OpenEdition.sol";
import {LPDAFactory} from "src/minters/LPDAFactory.sol";
import {LPDA} from "src/minters/LPDA.sol";
import {Escher721} from "src/Escher721.sol";

contract AuditTest is Test {
    address deployer;
    address creator;
    address buyer;

    FixedPriceFactory fixedPriceFactory;
    OpenEditionFactory openEditionFactory;
    LPDAFactory lpdaFactory;

    function setUp() public {
        deployer = makeAddr("deployer");
        creator = makeAddr("creator");
        buyer = makeAddr("buyer");

        vm.deal(buyer, 1e18);

        vm.startPrank(deployer);

        fixedPriceFactory = new FixedPriceFactory();
        openEditionFactory = new OpenEditionFactory();
        lpdaFactory = new LPDAFactory();

        vm.stopPrank();
    }
    
    function test_OpenEdition_buy_CancelSaleByRevokingRole() public {
        // Setup NFT and create sale
        vm.startPrank(creator);

        Escher721 nft = new Escher721();
        nft.initialize(creator, address(0), "Test NFT", "TNFT");

        uint24 startId = 0;
        uint72 price = 1e6;
        uint96 startTime = uint96(block.timestamp);
        uint96 endTime = uint96(block.timestamp + 1 hours);

        OpenEdition.Sale memory sale = OpenEdition.Sale(
            price, // uint72 price;
            startId, // uint24 currentId;
            address(nft), // address edition;
            startTime, // uint96 startTime;
            payable(creator), // address payable saleReceiver;
            endTime // uint96 endTime;
        );
        OpenEdition openSale = OpenEdition(openEditionFactory.createOpenEdition(sale));

        nft.grantRole(nft.MINTER_ROLE(), address(openSale));

        vm.stopPrank();

        // simulate we are in the middle of the sale duration
        vm.warp(startTime + 0.5 hours);

        // Now creator decides to pull out of the sale. Since he can't cancel the sale because it already started and he can't end the sale now because it hasn't finished, he revokes the minter role. This will cause the buy transaction to fail.
        vm.startPrank(creator);

        nft.revokeRole(nft.MINTER_ROLE(), address(openSale));

        vm.stopPrank();

        vm.startPrank(buyer);

        // Buyer can't call buy because sale contract can't mint tokens, the buy transaction reverts.
        uint256 amount = 1;
        vm.expectRevert();
        openSale.buy{value: price * amount}(amount);

        vm.stopPrank();

        // Now creator just needs to wait until sale ends
        vm.warp(endTime);

        vm.prank(creator);
        openSale.finalize();
    }
}
```

## Recommendation

One possibilty is to acknowledge the fact the creator has still control over the minting process and can arbitrarily decide when to cancel the sale. If this route is taken, then the recommendation would be to make the `cancel` function unrestricted.

Preminting the NFTs is not a solution because of high gas costs and the fact that the amount of tokens to be sold is not known beforehand, it's determined by the actual amount sold during the sale.

A more elaborated solution that would require additional architecture changes is to prevent the revocation of the minting permissions if some conditions (defined by target sale contract) are met.
