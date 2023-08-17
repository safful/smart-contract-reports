## Tags

- bug
- 3 (High Risk)
- sponsor acknowledged
- selected for report

# [StandardPolicyERC1155.sol returns amount == 1 instead of amount == order.amount](https://github.com/code-423n4/2022-10-blur-findings/issues/666) 

# Lines of code

https://github.com/code-423n4/2022-10-blur/blob/main/contracts/matchingPolicies/StandardPolicyERC1155.sol#L12-L36
https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L154-L161


# Vulnerability details

## Impact

The ```canMatchMakerAsk``` and ```canMatchMakerBid``` functions in ```StandardPolicyERC1155.sol``` will only return 1 as the amount instead of the order.amount value. This value is then used in the ```_executeTokenTransfer``` call during the execution flow and leads to only 1 ERC1155 token being sent. A buyer matching an ERC1155 order wih amount > 1 would expect to receive amount of tokens if they pay the order's price. The seller, who might also expect more than 1 tokens to be sent, would have set the order's price to be for the amount of tokens and not just for 1 token.

The buyer would lose overspent ETH/WETH to the seller without receiving all tokens as specified in the order.

## Proof of Concept

[StandardPolicyERC1155.sol:canMatchMakerAsk](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/matchingPolicies/StandardPolicyERC1155.sol#L12-L36)

```solidity
    function canMatchMakerAsk(Order calldata makerAsk, Order calldata takerBid)
        external
        pure
        override
        returns (
            bool,
            uint256,
            uint256,
            uint256,
            AssetType
        )
    {
        return (
            (makerAsk.side != takerBid.side) &&
            (makerAsk.paymentToken == takerBid.paymentToken) &&
            (makerAsk.collection == takerBid.collection) &&
            (makerAsk.tokenId == takerBid.tokenId) &&
            (makerAsk.matchingPolicy == takerBid.matchingPolicy) &&
            (makerAsk.price == takerBid.price),
            makerAsk.price,
            makerAsk.tokenId,
            1,
            AssetType.ERC1155
        );
    }
```

The code above shows that ```canMatchMakerAsk``` only returns 1 as the amount. ```_executeTokenTransfer``` will then [call the executionDelegate's ```transferERC1155``` function with only amount 1](https://github.com/code-423n4/2022-10-blur/blob/main/contracts/BlurExchange.sol#L540), transferring only 1 token to the buyer.


Test code added to ```execution.test.ts```:

```typescript
    it('Only 1 ERC1155 received for order with amount > 1', async () => {
      await mockERC1155.mint(alice.address, tokenId, 10);
      sell = generateOrder(alice, {
        side: Side.Sell,
        tokenId,
        amount: 10,
        collection: mockERC1155.address,
        matchingPolicy: matchingPolicies.standardPolicyERC1155.address,
      });
      buy = generateOrder(bob, {
        side: Side.Buy,
        tokenId,
        amount: 10,
        collection: mockERC1155.address,
        matchingPolicy: matchingPolicies.standardPolicyERC1155.address,
      });
      sellInput = await sell.pack();
      buyInput = await buy.pack();

      await waitForTx(exchange.execute(sellInput, buyInput));

      // Buyer only receives 1 token
      expect(await mockERC1155.balanceOf(bob.address, tokenId)).to.be.equal(1);
      await checkBalances(
        aliceBalance,
        aliceBalanceWeth.add(priceMinusFee),
        bobBalance,
        bobBalanceWeth.sub(price),
        feeRecipientBalance,
        feeRecipientBalanceWeth.add(fee),
      );
    });
```

The test code above shows a sell order for an ERC1155 token with amount = 10 and a matching buy order. The ```execute``` function in ```BlurExchange.sol``` is called and the orders are matched but the buyer (bob) only receives 1 token instead of 10 despite paying the full price.

## Recommended Mitigation Steps

Policies used for ERC1155 tokens should return and consider the amount of tokens set for the order.