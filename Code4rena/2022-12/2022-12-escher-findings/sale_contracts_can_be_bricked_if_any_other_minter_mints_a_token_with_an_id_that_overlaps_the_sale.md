## Tags

- bug
- 2 (Med Risk)
- disagree with severity
- downgraded by judge
- primary issue
- selected for report
- sponsor disputed
- edited-by-warden
- M-10

# [Sale contracts can be bricked if any other minter mints a token with an id that overlaps the sale](https://github.com/code-423n4/2022-12-escher-findings/issues/390) 

# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPrice.sol#L66
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/OpenEdition.sol#L67
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L74


# Vulnerability details

In all of the three sale types of contracts, the `buy` function will mint tokens with sequential ids starting at a given configurable value.

If an external entity mints any of the token ids involved in the sale, then the buy procedure will fail since it will try to mint an already existing id. This can be the creator manually minting a token or another similar contract that creates a sale.

Taking the `FixedPrice` contract as the example, if any of the token ids between `sale.currentId + 1` and `sale.finalId` is minted by an external entity, then the `buy` process will be bricked since it will try to mint an existing token id. This happens in lines 65-67:

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/FixedPrice.sol#L65-L67

```solidity
for (uint48 x = sale_.currentId + 1; x <= newId; x++) {
    nft.mint(msg.sender, x);
}
```

The implementation of the `mint` function (OZ contracts) requires that the token id doesn't exist:

https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/token/ERC721/ERC721Upgradeable.sol#L293

```solidity
function _mint(address to, uint256 tokenId) internal virtual {
    require(to != address(0), "ERC721: mint to the zero address");
    require(!_exists(tokenId), "ERC721: token already minted");
    ...
```

## Impact

If any of the scenarios previously described happens, then the contract will be bricked since it will try to mint an already existing token and will revert the transaction. There is no way to update the token ids in an ongoing sale, which means that the buy function will always fail for any call.

This will also cause a loss of funds in the case of the `FixedPrice` and `LPDA` contracts since those two require that all token ids are sold before funds can be withdrawn.

## PoC

The following test reproduces the issue using the `FixedPrice` sale contract type.

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
    
    function test_FixedPrice_buy_MintBrickedContract() public {
        // Suppose there's another protocol or account that acts as a minter, it can even be another sale that overlaps ids.
        address otherMinter = makeAddr("otherMinter");

        // Setup NFT and create sale
        vm.startPrank(creator);

        Escher721 nft = new Escher721();
        nft.initialize(creator, address(0), "Test NFT", "TNFT");

        uint48 startId = 10;
        uint48 finalId = 15;
        uint96 price = 1;

        FixedPrice.Sale memory sale = FixedPrice.Sale(
            startId, // uint48 currentId;
            finalId, // uint48 finalId;
            address(nft), // address edition;
            price, // uint96 price;
            payable(creator), // address payable saleReceiver;
            uint96(block.timestamp) // uint96 startTime;
        );
        FixedPrice fixedSale = FixedPrice(fixedPriceFactory.createFixedSale(sale));

        nft.grantRole(nft.MINTER_ROLE(), address(fixedSale));
        nft.grantRole(nft.MINTER_ROLE(), otherMinter);

        vm.stopPrank();

        // Now, other minter mints at least one of the ids included in the sale
        vm.prank(otherMinter);
        nft.mint(makeAddr("receiver"), startId + 2);

        // Now buyer goes to sale and tries to buy the NFTs
        vm.startPrank(buyer);

        // The following will revert. The contract will be bricked since it will be impossible to advance with the ids from the sale.
        uint256 amount = finalId - startId;
        vm.expectRevert("ERC721: token already minted");
        fixedSale.buy{value: price * amount}(amount);

        vm.stopPrank();
    }
}
```

## Recommendation

The quick solution would be to mint the tokens to the sale contract, but that will require a potentially high gas usage if the sale involves a large amount of tokens.

Other alternatives would be to "reserve" a range of tokens to a particular minter (and validate that each one mints over their own range) or to have a single minter role at a given time, so there's just a single entity that can mint tokens at a given time.
