## Tags

- bug
- 2 (Med Risk)
- primary issue
- satisfactory
- selected for report

# [Transaction origin check in ROE Markets make Options positions opened by contract users impossible to reduce or close](https://github.com/code-423n4/2023-09-goodentry-mitigation-findings/issues/17) 

# Lines of code

https://github.com/GoodEntry-io/GoodEntryMarkets/blob/2e3d23016fadb45e188716d772cec7c2096fae01/contracts/protocol/lendingpool/LendingPool.sol.0x20#L492
https://github.com/GoodEntry-io/ge/blob/c7c7de57902e11e66c8186d93c5bb511b53a45b8/contracts/PositionManager/OptionsPositionManager.sol#L386
https://github.com/GoodEntry-io/ge/blob/c7c7de57902e11e66c8186d93c5bb511b53a45b8/contracts/PositionManager/OptionsPositionManager.sol#L387
https://github.com/GoodEntry-io/ge/blob/c7c7de57902e11e66c8186d93c5bb511b53a45b8/contracts/PositionManager/OptionsPositionManager.sol#L412


# Vulnerability details

This issue was present in the original contest but I did not notice it as I did not have time to review OptionsPositionManager.

The Roe Markets `LendingPool.sol` that OptionsPositionManager uses is a modified version of Aave V2 with an added `PMTransfer` functionality, that is used by OptionsPositionManager when closing or reducing positions.

This `PMTransfer` only works when the user whose position is being operated on is in soft liquidation, or when the user initiated the transaction themselves:
```Solidity
    if (tx.origin != user) {
      (,,,, uint256 healthFactor) = GenericLogic.calculateUserAccountData(
        user,
        _reserves,
        _usersConfig[user],
        _reservesList,
        _reservesCount,
        _addressesProvider.getPriceOracle()
        );
      require(healthFactor <= softLiquidationThreshold, "Not initiated by user");
```

However, when positions are opened, OptionsPositionManager attributes debt to `user = msg.sender`.

```Solidity
  function buyOptions(
    // ...
  )
    external
  {
    // ...
    LP.flashLoan( address(this), options, amounts, flashtype, msg.sender, params, 0);
  }
```

This means that any user (EOA or contract) can open option positions - only for themselves - but only EOAs are materially able to close these positions.

## Impact
User interacting with OptionsPositionManager via a contract will be forced to stay into their positions until defaulting; only then, they can pass the check in `PMTransfer` and liquidate the position.

## Proof of Concept
Setting up is fairly simple in terms of steps, but requires interaction with a real Roe Markets lending pool (i.e. would work with a mainnet fork):
- Have a contract provide a generous amount of liquidity to the lending pool
- Have the contract open a relatively little leveraged position through OptionsPositionManager's `buyOptions` function. Little enough to not be in liquidation territory
- Have the contract close the position through OptionsPositionManager's `close`
- The `close` call will revert, having the user stuck in their position, accumulating debt until liquidation

```Solidity
  function testNotInitiatedByUser() public {
    // we have two addresses, the contractCaller (EOA), and theContract that interacts with the protocol
    address contractCaller = address(uint160(uint256(keccak256("contractCaller"))));
    address theContract = address(uint160(uint256(keccak256("theContract"))));

    // theContract has some collateral in the lending pool, so it is able to borrow assets
    changePrank(tokenWhale);
    WETH.transfer(theContract, 20e18);

    changePrank(operator);
    range.transfer(theContract, 1e18);

    changePrank(theContract);
    range.approve(address(RoeWethUsdcLP), 1e18);
    RoeWethUsdcLP.deposit(address(range), 1e18, theContract, 0);
    RoeWethUsdcLP.setUserUseReserveAsCollateral(address(range), true);

    // msg.sender different from tx.origin
    changePrank(theContract, contractCaller);

    // can open a position
    ICreditDelegationToken(0xB19Dd5DAD35af36CF2D80D1A9060f1949b11fCb0)
        .approveDelegation(address(optionsPM), type(uint256).max);
 
    address[] memory options = new address[](1);
    options[0] = address(range);
 
    uint256[] memory amounts = new uint256[](1);
    amounts[0] = 0.0001e18;
 
    address[] memory sourceSwap = new address[](1);
    sourceSwap[0] = address(USDC);

    optionsPM.buyOptions(
        0,
        options,
        amounts,
        sourceSwap
    );

    // simulate an always healthy position by changing the soft liquidation threshold to something really small
    vm.store(address(RoeWethUsdcLP), bytes32(uint256(0x3e)), bytes32(uint256(1)));

    // the position can't be closed 😱 - ROE markets' PMTransfer function does not allow that!
    WETH.approve(address(optionsPM), type(uint256).max);
    
    vm.expectRevert("Not initiated by user");
    optionsPM.close(0, 
        theContract,
        address(range), 
        0.0001e18,
        address(WETH));
  }
```


## Tools Used
Code review, Foundry

## Recommended Mitigation Steps
- Do not allow options positions to be opened to contracts
- Possibly, add an argument "onBehalfOf" so that contracts can still open positions for EOAs




## Assessed type

Invalid Validation