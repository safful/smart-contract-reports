## Tags

- bug
- 3 (High Risk)
- primary issue
- satisfactory
- selected for report
- edited-by-warden
- H-20

# [Deadlock in valuts with underlying token with less then 18 decimals](https://github.com/code-423n4/2023-01-astaria-findings/issues/72) 

# Lines of code

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/WithdrawProxy.sol#L271-L274


# Vulnerability details

## Impact
If underlying token for the vault would have less then 18 decimals, then after liquidation there would be no way to process epoch, because `claim` function in `WithdrawProxy.sol` would revert, this would lock all user out of their funds both in vault and in withdraw proxy. Alternatively, if there is more then 18 decimals, claim would left much less funds then needed for withdraw, resulting in withdrawers losing funds. 
To make report more concise, I would focus on tokens with less then 18 decimals, because they are much more frequent. For example, WBTC have 8 decimals and most stablecoins have 6.

## Why is this happening
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L314-L316
this part making sure that withdraw ratio are always stored in 1e18 scale.
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/WithdrawProxy.sol#L271-L274
but here, we are not transforming it into token decimals scale. `transferAmount` would be oders of magnitudes larger then balance
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/WithdrawProxy.sol#L277
then, here we would have underflow of `balance` value
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/WithdrawProxy.sol#L281
and finally, here function would revert. 

https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/WithdrawProxy.sol#L156
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/PublicVault.sol#L299
because `PublicVault.sol` need `claim` to proccess epoch, and `WithdrawProxy.sol` unlocks funds only after `claim`, it will result in deadlock of the whole system.

## Proof of Concept
First, creating token with 8 decimals:

    contract Token8Decimals is ERC20{
    constructor() ERC20("TEST", "TEST", 8) {}

    function mint(address to, uint amount) public{
        _mint(to, amount);
    }
    }

Second, I changed `_bid` function in `TestHelpers.t.sol` contract, so it could take token address as a last parameter, and use it instead of WETH.
Then, here is modified "testLiquidation5050Split" test:

    function testLiquidation5050Split() public {
    TestNFT nft = new TestNFT(2);
    _mintNoDepositApproveRouter(address(nft), 5);
    address tokenContract = address(nft);
    uint256 tokenId = uint256(1);

    Token8Decimals token = new Token8Decimals();

    // create a PublicVault with a 14-day epoch
    vm.startPrank(strategistOne);
    //bps
    address publicVault = (ASTARIA_ROUTER.newPublicVault(
      14 days,
      strategistTwo,
      address(token),
      uint256(0),
      false,
      new address[](0),
      uint256(0)
    ));
    vm.stopPrank();

    uint amountToLend = 10**8 * 1000;
    token.mint(address(1), amountToLend);
    vm.startPrank(address(1));
    token.approve(address(TRANSFER_PROXY), amountToLend);

    ASTARIA_ROUTER.depositToVault(
      IERC4626(publicVault),
      address(1),
      amountToLend,
      uint256(0)
    );
    vm.stopPrank();
    
    ILienToken.Details memory lien = standardLienDetails;
    lien.liquidationInitialAsk = amountToLend*2;

    (, ILienToken.Stack[] memory stack1) = _commitToLien({
      vault: publicVault,
      strategist: strategistOne,
      strategistPK: strategistOnePK,
      tokenContract: tokenContract,
      tokenId: tokenId,
      lienDetails: lien,
      amount: amountToLend/4,
      isFirstLien: true
    });

    uint256 collateralId = tokenContract.computeId(tokenId);

    _signalWithdraw(address(1), publicVault);

    WithdrawProxy withdrawProxy = PublicVault(publicVault).getWithdrawProxy(
      PublicVault(publicVault).getCurrentEpoch()
    );

    skip(14 days);

    OrderParameters memory listedOrder1 = ASTARIA_ROUTER.liquidate(
      stack1,
      uint8(0)
    );

    token.mint(bidder, amountToLend);
    _bid(Bidder(bidder, bidderPK), listedOrder1, amountToLend/2, address(token));
    vm.warp(withdrawProxy.getFinalAuctionEnd());
    emit log_named_uint("finalAuctionEnd", block.timestamp);
    PublicVault(publicVault).processEpoch();

    skip(13 days);

    withdrawProxy.claim();
    }

`withdrawProxy.claim();` at the last line would revert

## Recommended Mitigation Steps
https://github.com/code-423n4/2023-01-astaria/blob/1bfc58b42109b839528ab1c21dc9803d663df898/src/WithdrawProxy.sol#L273
change this line to `10**18`

## Severity
I think this is high risk, because
1) There are high demand for stablecoin denominated vaults, and Astaria are designed to support that
2) This bug is sneaky, there could be many epochs before first liquidation that would trigger the deadlock
3) ALL funds would be lost, which is catastrophic