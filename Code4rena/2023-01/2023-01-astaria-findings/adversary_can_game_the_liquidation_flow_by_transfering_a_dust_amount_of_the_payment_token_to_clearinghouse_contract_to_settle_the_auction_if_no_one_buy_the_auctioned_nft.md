## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report
- M-30

# [Adversary can game the liquidation flow by transfering a dust amount of the payment token to ClearingHouse contract to settle the auction if no one buy the auctioned NFT](https://github.com/code-423n4/2023-01-astaria-findings/issues/112) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/ClearingHouse.sol#L169
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/ClearingHouse.sol#L123


# Vulnerability details

## Impact

Adversary can game the liquidation flow by transfering a dust amount of the payment token to ClearingHouse contract to settle the auction

## Proof of Concept

The function ClearingHouse#safeTransferFrom is meant to settle the auction but the function severely lack of access control

```solidity
  function safeTransferFrom(
    address from, // the from is the offerer
    address to,
    uint256 identifier,
    uint256 amount,
    bytes calldata data //empty from seaport
  ) public {
    //data is empty and useless
    _execute(from, to, identifier, amount);
  }
```

which calls:

```solidity
  function _execute(
    address tokenContract, // collateral token sending the fake nft
    address to, // buyer
    uint256 encodedMetaData, //retrieve token address from the encoded data
    uint256 // space to encode whatever is needed,
  ) internal {
    IAstariaRouter ASTARIA_ROUTER = IAstariaRouter(_getArgAddress(0)); // get the router from the immutable arg

    ClearingHouseStorage storage s = _getStorage();
    address paymentToken = bytes32(encodedMetaData).fromLast20Bytes();

    uint256 currentOfferPrice = _locateCurrentAmount({
      startAmount: s.auctionStack.startAmount,
      endAmount: s.auctionStack.endAmount,
      startTime: s.auctionStack.startTime,
      endTime: s.auctionStack.endTime,
      roundUp: true //we are a consideration we round up
    });
    uint256 payment = ERC20(paymentToken).balanceOf(address(this));

    require(payment >= currentOfferPrice, "not enough funds received");

    uint256 collateralId = _getArgUint256(21);
    // pay liquidator fees here

    ILienToken.AuctionStack[] storage stack = s.auctionStack.stack;

    uint256 liquidatorPayment = ASTARIA_ROUTER.getLiquidatorFee(payment);

    ERC20(paymentToken).safeTransfer(
      s.auctionStack.liquidator,
      liquidatorPayment
    );

    ERC20(paymentToken).safeApprove(
      address(ASTARIA_ROUTER.TRANSFER_PROXY()),
      payment - liquidatorPayment
    );

    ASTARIA_ROUTER.LIEN_TOKEN().payDebtViaClearingHouse(
      paymentToken,
      collateralId,
      payment - liquidatorPayment,
      s.auctionStack.stack
    );

    if (ERC20(paymentToken).balanceOf(address(this)) > 0) {
      ERC20(paymentToken).safeTransfer(
        ASTARIA_ROUTER.COLLATERAL_TOKEN().ownerOf(collateralId),
        ERC20(paymentToken).balanceOf(address(this))
      );
    }
    ASTARIA_ROUTER.COLLATERAL_TOKEN().settleAuction(collateralId);
  }
```

We can look into the liquidation flow:

first, the liquidate function is called in AstariaRouter.sol

```solidity
  function liquidate(ILienToken.Stack[] memory stack, uint8 position)
    public
    returns (OrderParameters memory listedOrder)
  {
    if (!canLiquidate(stack[position])) {
      revert InvalidLienState(LienState.HEALTHY);
    }

    RouterStorage storage s = _loadRouterSlot();
    uint256 auctionWindowMax = s.auctionWindow + s.auctionWindowBuffer;

    s.LIEN_TOKEN.stopLiens(
      stack[position].lien.collateralId,
      auctionWindowMax,
      stack,
      msg.sender
    );
    emit Liquidation(stack[position].lien.collateralId, position);
    listedOrder = s.COLLATERAL_TOKEN.auctionVault(
      ICollateralToken.AuctionVaultParams({
        settlementToken: stack[position].lien.token,
        collateralId: stack[position].lien.collateralId,
        maxDuration: auctionWindowMax,
        startingPrice: stack[0].lien.details.liquidationInitialAsk,
        endingPrice: 1_000 wei
      })
    );
  }
```

then function auctionVault is called in CollateralToken

```solidity
function auctionVault(AuctionVaultParams calldata params)
	external
	requiresAuth
returns (OrderParameters memory orderParameters)
{
CollateralStorage storage s = _loadCollateralSlot();

uint256[] memory prices = new uint256[](2);
prices[0] = params.startingPrice;
prices[1] = params.endingPrice;
orderParameters = _generateValidOrderParameters(
  s,
  params.settlementToken,
  params.collateralId,
  prices,
  params.maxDuration
);

_listUnderlyingOnSeaport(
  s,
  params.collateralId,
  Order(orderParameters, new bytes(0))
);
}
```

the function _generateValidOrderParameters is important:

```solidity

  function _generateValidOrderParameters(
    CollateralStorage storage s,
    address settlementToken,
    uint256 collateralId,
    uint256[] memory prices,
    uint256 maxDuration
  ) internal returns (OrderParameters memory orderParameters) {
    OfferItem[] memory offer = new OfferItem[](1);

    Asset memory underlying = s.idToUnderlying[collateralId];

    offer[0] = OfferItem(
      ItemType.ERC721,
      underlying.tokenContract,
      underlying.tokenId,
      1,
      1
    );

    ConsiderationItem[] memory considerationItems = new ConsiderationItem[](2);
    considerationItems[0] = ConsiderationItem(
      ItemType.ERC20,
      settlementToken,
      uint256(0),
      prices[0],
      prices[1],
      payable(address(s.clearingHouse[collateralId]))
    );
    considerationItems[1] = ConsiderationItem(
      ItemType.ERC1155,
      s.clearingHouse[collateralId],
      uint256(uint160(settlementToken)),
      prices[0],
      prices[1],
      payable(s.clearingHouse[collateralId])
    );

    orderParameters = OrderParameters({
      offerer: s.clearingHouse[collateralId],
      zone: address(this), // 0x20
      offer: offer,
      consideration: considerationItems,
      orderType: OrderType.FULL_OPEN,
      startTime: uint256(block.timestamp),
      endTime: uint256(block.timestamp + maxDuration),
      zoneHash: bytes32(collateralId),
      salt: uint256(blockhash(block.number)),
      conduitKey: s.CONDUIT_KEY, // 0x120
      totalOriginalConsiderationItems: considerationItems.length
    });
  }
```

note the first consideration item:

```solidity
ConsiderationItem[] memory considerationItems = new ConsiderationItem[](2);
considerationItems[0] = ConsiderationItem(
  ItemType.ERC20,
  settlementToken,
  uint256(0),
  prices[0],
  prices[1],
  payable(address(s.clearingHouse[collateralId]))
);
```

prices[0] is the initialLiquidationPrice the starting price, prices[1] is the ending price, which is 1000 WEI. this means if no one buy the dutch auction, the price will fall to 1000 WEI.

```solidity
ClearingHouseStorage storage s = _getStorage();
address paymentToken = bytes32(encodedMetaData).fromLast20Bytes();

uint256 currentOfferPrice = _locateCurrentAmount({
  startAmount: s.auctionStack.startAmount,
  endAmount: s.auctionStack.endAmount,
  startTime: s.auctionStack.startTime,
  endTime: s.auctionStack.endTime,
  roundUp: true //we are a consideration we round up
});
uint256 payment = ERC20(paymentToken).balanceOf(address(this));

require(payment >= currentOfferPrice, "not enough funds received");
```

In the code above, note that the code check if the current balance of the payment token is larger than currentOfferPrice computed by _locateCurrentAmount, this utility function comes from seaport.

https://github.com/ProjectOpenSea/seaport/blob/f402dac8b3faabdb8420d31d46759f47c9d74b7d/contracts/lib/AmountDeriver.sol#L38

If there is no one buy the dutch auction, the price will drop to until 1000 WEI, which means _locateCurrentAmount returns lower and lower price.

In normal flow, if no one buy the dutch auction and cover the outstanding debt, the NFT can be claimabled by liquidator. The liquidator can try to sell NFT again to cover the debt and loss for lenders,

However, if no one wants to buy the dutch auction and the _locateCurrentAmount is low enough, for example, 10000 WEI, an adversary can transfer 10001 WEI of ERC20 payment token to the ClearingHouse contract 

Then call safeTransferFrom to settle the auction.

the code below will execute

```solidity
uint256 payment = ERC20(paymentToken).balanceOf(address(this));

require(payment >= currentOfferPrice, "not enough funds received");
```
.
beceause the airdropped ERC20 balance make the payment larger than currentOfferPrice

In this case, the adversary blocks the liquidator from claiming the not-auctioned liquidated NFT.

the low amount of payment 10001 WEI is not likely to cover the outstanding debt and the lender has to bear the loss.

## Tools Used

Manual Review

## Recommended Mitigation Steps

We recommend the protocol validate the caller of the safeTransferFrom in ClearingHouse is the seaport / conduict contract and check that when the auction is settled, the NFT ownership changed and the Astaria contract does not hold the NFT any more.