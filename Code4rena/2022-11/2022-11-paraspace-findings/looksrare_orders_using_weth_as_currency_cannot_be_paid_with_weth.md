## Tags

- bug
- 2 (Med Risk)
- primary issue
- selected for report
- M-11

# [LooksRare orders using WETH as currency cannot be paid with WETH](https://github.com/code-423n4/2022-11-paraspace-findings/issues/344) 

# Lines of code

https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/marketplaces/LooksRareAdapter.sol#L51-L54
https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/libraries/logic/MarketplaceLogic.sol#L564-L565


# Vulnerability details

## Impact
Users won't be able to buy NFTs from LooksRare via the Paraspace Marketplace and pay with WETH when `MakerOrder` currency is set to WETH.
## Proof of Concept
The Paraspace Marketplace allows users to buy NFTs from third-party marketplaces (LooksRare, Seaport, X2Y2) using funds borrowed from Paraspace. The mechanism of buying tokens requires a `MakerOrder`: a data structure that's created by the seller and posted on a third-party exchange and that contains all the information about the order. Besides other fields, `MakerOrder` contains the `currency` field, which sets the currency the buyer is willing to receive payment in ([OrderTypes.sol#L20](https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/dependencies/looksrare/contracts/libraries/OrderTypes.sol#L20)):
```solidity
struct MakerOrder {
    ...
    address currency; // currency (e.g., WETH)
    ...
}
```

While WETH is supported by LooksRare as a currency of orders, the LooksRare adapter of Paraspace converts it to the native (e.g. ETH) currency: if order's currency is WETH, the currency of consideration is set to the native currency ([LooksRareAdapter.sol#L49-L62](https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/misc/marketplaces/LooksRareAdapter.sol#L49-L62)):
```solidity
ItemType itemType = ItemType.ERC20;
address token = makerAsk.currency;
if (token == weth) {
    itemType = ItemType.NATIVE;
    token = address(0);
}
consideration[0] = ConsiderationItem(
    itemType,
    token,
    0,
    makerAsk.price, // TODO: take minPercentageToAsk into account
    makerAsk.price,
    payable(takerBid.taker)
);
```

When users call the `buyWithCredit` function, they provide credit parameters: token address and amount (besides others) ([PoolMarketplace.sol#L71-L76](https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/pool/PoolMarketplace.sol#L71-L76), [DataTypes.sol#L296-L303](https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/libraries/types/DataTypes.sol#L296-L303)):
```solidity
function buyWithCredit(
    bytes32 marketplaceId,
    bytes calldata payload,
    DataTypes.Credit calldata credit,
    uint16 referralCode
) external payable virtual override nonReentrant {
```

```solidity
struct Credit {
    address token;
    uint256 amount;
    bytes orderId;
    uint8 v;
    bytes32 r;
    bytes32 s;
}
```

Deep inside the `buyWithCredit` function, `isETH` flag is set when the credit token specified by user is `address(0)` ([MarketplaceLogic.sol#L564-L566](https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/libraries/logic/MarketplaceLogic.sol#L564-L566)):
```solidity
vars.isETH = params.credit.token == address(0);
vars.creditToken = vars.isETH ? params.weth : params.credit.token;
vars.creditAmount = params.credit.amount;
```

Finally, before giving a credit, consideration type and credit token are checked ([MarketplaceLogic.sol#L388-L392](https://github.com/code-423n4/2022-11-paraspace/blob/c6820a279c64a299a783955749fdc977de8f0449/paraspace-core/contracts/protocol/libraries/logic/MarketplaceLogic.sol#L388-L392)):
```solidity
require(
    item.itemType == ItemType.ERC20 ||
        (vars.isETH && item.itemType == ItemType.NATIVE),
    Errors.INVALID_ASSET_TYPE
);
```

This is what happens when a user tries to buy an NFT from LooksRare and pays with WETH as the maker order requires:
1. User calls `buyWithCredit` and sets credit token to WETH.
1. The currency of the consideration is changed to the native currency, and the consideration token is set to `address(0)`.
1. `var.isETH` is not set since the credit token is WETH, not `address(0)`.
1. The consideration type and credit token check fails because the type of the consideration is `NATIVE` but `var.isETH` is not set.
1. As a result, the call to `buyWithCredit` reverts and the user cannot buy a token while correctly using the API.

```js
it("fails when trying to buy a token on LooksRare with WETH [AUDIT]", async () => {
  const {
    doodles,
    pool,
    weth,
    users: [maker, taker, middleman],
  } = await loadFixture(testEnvFixture);
  const payNowNumber = "8";
  const creditNumber = "2";
  const payNowAmount = await convertToCurrencyDecimals(
    weth.address,
    payNowNumber
  );
  const creditAmount = await convertToCurrencyDecimals(
    weth.address,
    creditNumber
  );
  const startAmount = payNowAmount.add(creditAmount);
  const nftId = 0;
  // mint WETH to offer
  await mintAndValidate(weth, payNowNumber, taker);
  // middleman supplies DAI to pool to be borrowed by offer later
  await supplyAndValidate(weth, creditNumber, middleman, true);
  // maker mint mayc
  await mintAndValidate(doodles, "1", maker);
  // approve
  await waitForTx(
    await weth.connect(taker.signer).approve(pool.address, payNowAmount)
  );

  await expect(
    executeLooksrareBuyWithCredit(
      doodles,
      weth as MintableERC20,
      startAmount,
      creditAmount,
      nftId,
      maker,
      taker
    )
  ).to.be.revertedWith('93'); // invalid asset type for action.
});
```
## Tools Used
Manual review
## Recommended Mitigation Steps
Consider removing the WETH to NATIVE conversion in the LooksRare adapter. Alternatively, consider converting WETH to ETH seamlessly, without forcing users to send ETH instead of WETH when maker order requires WETH.